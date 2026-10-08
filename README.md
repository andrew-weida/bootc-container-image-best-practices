# Bootc Container Image Best Practices

A comprehensive guide for building bootable container (bootc) images and migrating applications into them. Compiled from the official bootc documentation, Red Hat RHEL 9 and RHEL 10 Image Mode documentation, and the Red Hat Community of Practice demo repository.

> **Version scope**: Examples target **RHEL 10 image mode** unless noted. Where RHEL 9 behaves differently or lacks a feature, a **RHEL 9 vs RHEL 10** callout marks it. Guidance that comes from the upstream bootc project but is *not* present in Red Hat's RHEL 9 or RHEL 10 image mode guides is marked **Upstream only** — it may work, but it is outside what Red Hat documents.

---

## What Is bootc?

bootc provides **transactional, in-place operating system updates using OCI/Docker container images**. It uses the standard OCI image format as a transport and delivery mechanism for full operating system updates, including the Linux kernel.

Key distinctions from regular containers:

- A bootc image includes the **kernel, initrd, bootloader, and firmware** needed to boot bare metal or VMs
- After deployment, the system **does not run as a container**. systemd acts as PID 1 as usual
- The underlying storage engine is **ostree**, which has powered stable OS updates for years
- OCI directives like `ENTRYPOINT`, `CMD`, `ENV`, `EXPOSE`, and `USER` are **ignored when installed to a system** but still respected when run via `podman` or `docker`

---

## Filesystem Semantics

Understanding the filesystem layout is the single most important concept for building correct bootc images.

### Build Time vs. Deploy Time

| Phase | Filesystem behavior |
|-------|-------------------|
| **Build time** (in Containerfile) | Entire filesystem is fully writable |
| **Deployed system** (booted VM/bare metal) | `/usr` is read-only, `/etc` is writable with merge, `/var` is writable and persistent |

### Directory Roles

| Directory | Deployed behavior | Key details |
|-----------|------------------|-------------|
| `/usr` | **Read-only** (immutable) | All OS content belongs here. `/bin`, `/lib`, `/sbin` are symlinks into `/usr`. |
| `/etc` | **Writable**, persistent with 3-way merge | On update, ostree merges your local changes with the new image's defaults. Metadata changes (uid, gid, xattrs) count as modifications. |
| `/var` | **Writable**, persistent, **no merge on update** | Content from the image is unpacked **only on initial install**. Subsequent image updates never overwrite `/var`. Behaves like a Docker `VOLUME`. |
| `/opt` | **Read-only** (composefs is enabled by default) | A plain directory, not a symlink into `/var` — installable at build time, not writable at runtime. Third-party software expecting to write here needs special handling (see below). |
| `/run`, `/proc`, `/sys` | API filesystems | Shipping content here in images is **not supported**. |
| `/sysroot` | Physical root | The actual host filesystem; deployment roots are subdirectories. The booted system resembles a chroot into a *deployment root*, located via the `ostree=` kernel argument. |

### The `/var` Rule

This is the most common source of confusion:

> Content written to `/var` during the container build is only applied during the **initial installation** of the image. Updates to `/var` in later image versions are silently ignored on already-deployed systems.

**Always use `systemd-tmpfiles` or `StateDirectory=` to ensure `/var` subdirectories exist**, rather than relying on `mkdir` in the Containerfile.

### The `/etc` Merge Strategy

During upgrades, ostree performs a 3-way merge at shutdown time:

1. New image's default `/etc` serves as the base
2. The diff between your current `/etc` and the previous image's default is computed
3. Your local modifications are reapplied on top of the new defaults

If you need no per-machine state in `/etc`, enable transient mode:

```ini
# /usr/lib/ostree/prepare-root.conf
[etc]
transient = true
```

### Handling `/opt`

Because composefs is in use, `/opt` is a plain directory inside the immutable image (not symlinks into `/var`). Third-party software can therefore be *installed* into `/opt` at build time — it just cannot *write* there at runtime.

Approaches, in order of preference:

1. **Symlink the writable subdirs into `/var`** — maximum immutability, and the pattern Red Hat documents:

   ```dockerfile
   RUN rmdir /opt/exampleapp/logs && ln -sr /var/log/exampleapp /opt/exampleapp/logs
   ```

2. **`BindPaths=` in the systemd unit** — redirect at runtime without altering the image layout:

   ```ini
   [Service]
   BindPaths=/var/log/exampleapp:/opt/exampleapp/logs
   ```

3. **Transient read-only root (`transient-ro`)** — an overlayfs on `/` whose upper layer is mounted read-only. A privileged process can create top-level directories or symlinks inside a private mount namespace while every other process still sees an immutable root. Changes last for the current boot only.

   Use it when you need an absolute top-level path that cannot exist at build time: a bind-mount target, a platform-specific mountpoint such as `/users`, or anything decided after deployment. For content that must persist, symlink into `/var` instead.

   Setup — the `prepare-root.conf` key, initramfs regeneration, build capabilities, the `LIBMOUNT_FORCE_MOUNT2=always` requirement, and a worked systemd unit — is covered in Red Hat's documentation: [RHEL 10](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/creating-root-level-directories-and-symlinks-at-runtime-with-image-mode-for-rhel) · [RHEL 9](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/creating-root-level-directories-and-symlinks-at-runtime-with-image-mode-for-rhel).

---

## Containerfile Patterns

### Basic Structure

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:10.2

# Install packages
RUN dnf -y install httpd vim && dnf clean all

# Enable services (no --now flag!)
RUN systemctl enable httpd

# Add configuration files
COPY files/my-config.conf /etc/httpd/conf.d/my-config.conf

# Add custom content to /usr (immutable at runtime)
COPY files/index.html /usr/share/www/html/index.html

# Declare container image is a bootc-compatible operating system image
LABEL containers.bootc=1
LABEL ostree.bootable=1
```

### OCI Directives: What Gets Ignored

When a bootc image is **installed to a system**, these directives are ignored:

| Directive | Why ignored | What to do instead |
|-----------|-------------|-------------------|
| `ENTRYPOINT` / `CMD` | systemd is PID 1 | Set `CMD /sbin/init` for container testing compatibility |
| `ENV` | No container runtime to inject env vars | Configure via systemd unit `Environment=` or drop-in files |
| `USER` | Not a container process model | Configure individual services to run as unprivileged users |
| `EXPOSE` | No container networking stack | Configure firewall rules via firewalld XML files |

These directives **are** respected when running the image via `podman run` or `docker run` for testing.

### Multi-Stage Builds

Use multi-stage builds to keep build tools out of the deployed image:

```dockerfile
FROM registry.access.redhat.com/ubi10/go-toolset:latest AS builder
COPY . /opt/app-root/src/
RUN go build -o /opt/app-root/src/myapp

FROM registry.redhat.io/rhel10/rhel-bootc:10.2
COPY --from=builder /opt/app-root/src/myapp /usr/local/bin/myapp
COPY myapp.service /usr/lib/systemd/system/myapp.service
RUN systemctl enable myapp
```

The deployment image should include **only the application and its required runtime**, never build tools.

`/usr/local` is a plain directory in bootc base images (unlike RHEL CoreOS, where it is a symlink into `/var`), so it is writable at build time and immutable at runtime — a valid place for custom binaries.

### Relocating Content Out of `/var`

Since `/var` is not updated after initial install, move application content that ships with the image into `/usr`:

```dockerfile
# Apache example: move web root from /var to /usr
RUN dnf -y install httpd && \
    systemctl enable httpd && \
    mv /var/www /usr/share/www && \
    sed -ie 's,/var/www,/usr/share/www,' /etc/httpd/conf/httpd.conf

COPY files/index.html /usr/share/www/html/index.html
```

This ensures web content updates with each new image version, rather than being frozen at first install.

### Ensuring `/var` Directories Exist

Use `systemd-tmpfiles` to create required `/var` subdirectories at boot:

```dockerfile
# Create a tmpfiles.d config for MariaDB
RUN cat > /usr/lib/tmpfiles.d/mariadb-dirs.conf << 'EOF'
d /var/lib/mysql 0755 mysql mysql -
d /var/log/mariadb 0755 mysql mysql -
EOF
```

Or in a systemd unit, use `StateDirectory=`:

```ini
[Service]
StateDirectory=myapp
# Creates /var/lib/myapp owned by the service user
```

---

## Migrating Applications to bootc

### The Build Environment Is Not a Running System

The most critical difference: **systemd is not running during image builds**. There is no D-Bus, no running services, no hardware context.

| Aspect | Traditional RPM install | bootc image build |
|--------|------------------------|-------------------|
| systemd | Active, D-Bus available | **Not running** |
| Network | Full access | Available but discouraged |
| Interactive input | TTY can be available | **No TTY; prompts hang** |
| Hardware | Detects real hardware | Detects build host/container |
| `/run/systemd/system` | Exists | **Does not exist** |

### Common RPM Scriptlet Failures and Fixes

#### `systemctl enable --now`

The `--now` flag tries to start the service immediately, which fails without systemd.

```dockerfile
# WRONG
RUN systemctl enable --now myservice

# CORRECT
RUN systemctl enable myservice
```

Or create the symlink manually:

```dockerfile
RUN ln -s ../myservice.service /usr/lib/systemd/system/multi-user.target.wants/
```

#### `systemctl restart` in scriptlets

Remove entirely. Services start at first boot if enabled.

#### Scriptlets checking `/run/systemd/system`

This directory doesn't exist during builds. Copy unit files explicitly:

```dockerfile
RUN cp /usr/lib/myapp/myservice.service /usr/lib/systemd/system/myservice.service
RUN systemctl enable myapp.service
```

#### `useradd` / `groupadd` in scriptlets

UID/GID allocation during builds may differ across rebuilds. Use `systemd-sysusers`:

```dockerfile
RUN cat > /usr/lib/sysusers.d/myapp.conf << 'EOF'
u myappuser 900 "My Application Service Account" /var/lib/myapp /sbin/nologin
EOF
```

#### `mkdir` under `/var`

Directories created at build time in `/var` only apply during initial installation. Use `systemd-tmpfiles`:

```dockerfile
RUN cat > /usr/lib/tmpfiles.d/myapp.conf << 'EOF'
d /var/log/myapp 0755 myappuser myappuser -
d /var/lib/myapp 0750 myappuser myappuser -
EOF
```

#### Interactive prompts (`read`, `dialog`)

Defer to a first-boot systemd service:

```ini
[Unit]
Description=First-boot configuration for my application
ConditionFirstBoot=yes
Before=myapp.service

[Service]
Type=oneshot
ExecStart=/usr/libexec/myapp/first-boot-setup.sh
RemainAfterExit=yes
StandardInput=tty
TTYPath=/dev/tty1

[Install]
WantedBy=multi-user.target
```

#### Host-specific configuration (hostname, IP)

Never generate at build time. Defer to a first-boot service or use cloud-init/Ignition.

#### PID files written to `/run`

Content in `/run` is lost every boot. Use `Type=notify` or `Type=exec` in systemd units instead of PID files.

#### `firewall-cmd --add-port`

firewalld isn't running at build time. Place XML configuration files instead:

```dockerfile
COPY myapp-firewall.xml /usr/lib/firewalld/services/myapp.xml
```

### RPM Adaptation Checklist

Before embedding any existing RPM in a bootc image, audit for:

- [ ] `systemctl` / `systemd` calls in `%pre` / `%post` scriptlets
- [ ] `useradd` / `groupadd` / `getent` commands (replace with `systemd-sysusers`)
- [ ] `mkdir` targeting `/var` (replace with `systemd-tmpfiles`)
- [ ] `read` / `dialog` interactive prompts (defer to first-boot service)
- [ ] Hostname or IP references (defer to first-boot or provisioning tool)
- [ ] Hard-coded writable paths in `/usr` or `/opt` (redirect to `/var` via symlinks or `BindPaths=`)
- [ ] SELinux contexts (add `RUN restorecon -R <path>` if needed)
- [ ] PID file writes to `/run` or `/var/run`
- [ ] `firewall-cmd` calls (replace with XML config files)
- [ ] Network service dependencies during install

Run `rpm -qlp <package>` and compare against files present in the built image to catch missing files.

---

## User and Group Management

### System Service Accounts

Use `systemd-sysusers` with explicit UIDs for reproducibility:

```dockerfile
RUN cat > /usr/lib/sysusers.d/myapp.conf << 'EOF'
u myappuser 900 "My App Service" /var/lib/myapp /sbin/nologin
g myappgroup 900
EOF
```

For service accounts that do not need a stable identity on disk, Red Hat recommends `DynamicUser=yes` in the unit as "significantly better than the pattern of allocating users or groups at package install time."

Base images built by rpm-ostree enable **`nss-altfiles`**, which splits users into `/usr/lib/passwd` and `/usr/lib/group` and pre-allocates system users to avoid UID/GID drift. Note the corollary: once `/etc/passwd` is modified locally, later image changes to it are no longer applied.

### Interactive Users

For demo/development, users can be created directly:

```dockerfile
RUN dnf -y install mkpasswd
RUN pass=$(mkpasswd --method=SHA-512 --rounds=4096 mypassword) && \
    useradd -m -G wheel myuser -p $pass
RUN echo "%wheel ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/wheel-sudo
```

Red Hat's documented equivalent is a `bootc-image-builder` config file rather than a Containerfile `useradd`:

```toml
[[customizations.user]]
name = "user"
password = "pass"
key = "ssh-rsa AAA... user@email.com"
groups = ["wheel"]
```

Passed with `--config /config.toml`. Only `name` is mandatory.

> ⚠️ Do not encrypt passwords or SSH keys with publicly-available private keys in generic images. Plain `useradd` also allocates UIDs/GIDs dynamically, which causes drift across rebuilds — use `systemd-sysusers` or `DynamicUser=yes` for anything that must stay stable.

Because `/home` is a symlink to `/var/home`, injecting `/var/home/someuser/.ssh/authorized_keys` into a later image version **will not reach already-installed systems**. If you want image-managed keys, pair a `tmpfs` `/home` with a tmpfiles.d entry in `/usr/lib/tmpfiles.d/`:

```
f~ /home/user/.ssh/authorized_keys 600 user user - <base64 encoded data>
```

For production, prefer provisioning tools:

- **cloud-init**: Inject users and SSH keys on first boot
- **Ignition**: Initramfs-stage provisioning (Fedora CoreOS style)
- **`bootc install --root-ssh-authorized-keys`**: Inject SSH keys at install time
- **systemd-firstboot**: Set locale, timezone, hostname

### Base Image Security

Base bootc images ship **without default passwords or SSH keys**. This is intentional. Configure authentication via provisioning, not baked into the image.

---

## Service Management

### Enabling Services

```dockerfile
# Standard enable
RUN systemctl enable httpd

# For cloud-init
RUN ln -s ../cloud-init.target /usr/lib/systemd/system/default.target.wants
```

### First-Boot Services

Use `ConditionFirstBoot=yes` for one-time setup tasks:

```ini
[Unit]
Description=One-time application setup
ConditionFirstBoot=yes

[Service]
Type=oneshot
ExecStart=/usr/libexec/myapp/setup.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

### Bound Container Images

Applications that run as containers on the bootc host can be **bound** to the system image, so the OS and its workloads version and update as one unit instead of drifting apart. Both forms use Quadlet files and are built into the image; they differ in where the application image data lives.

**Logically bound** — the image is *referenced* by the base image and pulled into the bootc-owned store during `bootc upgrade`, before the reboot. Use this for agents that must be running from early boot and must not drift from the host: log forwarders, monitoring exporters, configuration management, security agents. It avoids rebuilding the base image for an app-only change, but disconnected installs must mirror every referenced container.

**Physically bound** — the application containers and their Quadlets are *embedded* in the root filesystem at build time, shipping as one self-contained artifact. Use this when you need strict offline reliability or resilience to registry outages. The cost is that any app change requires rebuilding the full base image.

Setup for both — the Quadlet layout, the `bound-images.d/` symlink, the additional-image-store argument, and the `rhel-system-roles` build for physically bound images — is covered in Red Hat's documentation:

- Logically bound: [RHEL 10](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/building-and-managing-logically-bound-images) · [RHEL 9](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/building-and-managing-logically-bound-images)
- Physically bound: [RHEL 10](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/building-and-managing-physically-bound-images) · [RHEL 9](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/building-and-managing-physically-bound-images)

---

## Updates and Rollbacks

### How Updates Work

1. Build a new version of your container image
2. Push to your registry
3. On the deployed system, run `bootc upgrade` (or rely on automatic timer-based updates)
4. bootc fetches **only the changed layers** from the registry
5. A reboot applies the update (on RHEL 10, a soft reboot for non-kernel changes)

```bash
# Check whether an update is available. Does NOT download or stage anything.
sudo bootc upgrade --check

# Fetch and stage the update without touching the running system
sudo bootc upgrade
sudo reboot

# Fetch, stage, and reboot in one step
sudo bootc upgrade --apply
```

`bootc update` is an alias for `bootc upgrade`; both have the same effect. This guide uses `upgrade`, which is the spelling Red Hat's documentation uses.

> **RHEL 9 vs RHEL 10**: **soft reboot is RHEL 10 only.** On RHEL 10 you can skip firmware init and swap userspace in place:
>
> ```bash
> sudo bootc upgrade --soft-reboot=required --apply   # fail if a soft reboot is not possible
> sudo bootc upgrade --soft-reboot=auto --apply       # fall back to a full reboot
> ```
>
> The same flag works on `bootc switch` and `bootc rollback`. RHEL 9's systemd does not support soft reboot, and these commands will fail there. Soft reboot never applies kernel, driver, or kernel-argument changes — those always need a full reboot — and it does not reset `sysctl` settings. Confirm afterwards with `bootc status --verbose`.

#ƒ## Automatic Updates

Enabled by default via the `bootc-fetch-apply-updates.timer` systemd unit. To turn it off:

```bash
sudo systemctl mask bootc-fetch-apply-updates.timer
```

Or reschedule it with a drop-in placed at `/usr/lib/systemd/system/bootc-fetch-apply-updates.timer.d/updates.conf`:

```ini
[Timer]
# Clear the inherited timers, then run weekly
OnBootSec=
OnBootSec=1w
OnUnitInactiveSec=1w
```

Note that Red Hat rebuilds base images at least monthly, but **a new base image does not trigger an automatic rebuild of your derived images** — that is your pipeline's job.

### Rollbacks

bootc keeps the previous deployment. Boot into it via the bootloader menu, or:

```bash
sudo bootc rollback
sudo reboot
```

Any staged-but-unapplied upgrade is discarded. **If automatic updates are still enabled, the system will update itself again** — mask the timer first. If the problem you are rolling back from does not involve `/etc`, `bootc switch` back to the known-good tag is often the better tool.

### Image Switching

Switch to an entirely different image:

```bash
sudo bootc switch [--apply] quay.io/myorg/new-image:latest
```

`bootc switch` has the same effect as `bootc upgrade` apart from changing the image reference, and it preserves existing state in `/etc` and `/var` (host SSH keys, home directories).

### Offline / Air-Gapped Updates

`bootc upgrade` has no way to specify an alternate source, so use `bootc switch --transport` to repoint at local media, after which `bootc upgrade` keeps working against that location:

```bash
# On a connected host: copy the image onto removable media
skopeo copy --preserve-digests --all \
    docker://quay.io/myorg/my-image:latest oci:/mnt/usb/

# On the air-gapped host
sudo bootc switch --transport oci /mnt/usb
# or, after importing into local container storage
sudo bootc switch --transport containers-storage example.io/library/rhel-update:latest
```

### What Persists Across Updates

| Directory | Update behavior |
|-----------|----------------|
| `/usr` | Replaced entirely with new image content |
| `/etc` | 3-way merged (local changes preserved where possible) |
| `/var` | **Untouched** by updates |
| Kernel args | Preserved unless explicitly changed |

---

## Building and Deploying Disk Images

### bootc-image-builder

Converts container images into deployable disk formats:

| Format | Target platform |
|--------|----------------|
| `qcow2` | QEMU, OpenStack, libvirt |
| `ami` | AWS |
| `vmdk` | VMware vSphere |
| `raw` | Unformatted raw disk |
| `vhd` | Azure / Virtual PC |
| `gce` | Google Compute Engine |

### Requirements

- **Rootful Podman required**: rootless mode causes build failure
- Must run with `--privileged` and SELinux unconfined
- Shipped only as a container image — there is no RPM
- The tool cannot pull from remote registries itself. Either mount your local container storage and pass `--local`, or make the source image **public**
- ISO builds additionally require a subscribed host (or repository configuration injected via bind mounts)
- AMI builds require an existing S3 bucket and the `vmimport` service role on the AWS account

```bash
sudo podman login registry.redhat.io
sudo podman pull registry.redhat.io/rhel10/bootc-image-builder:latest

sudo podman run --rm --privileged --pull=newer \
    --security-opt label=type:unconfined_t \
    -v /var/lib/containers/storage:/var/lib/containers/storage \
    -v ./output:/output \
    -v ./config.toml:/config.toml:ro \
    registry.redhat.io/rhel10/bootc-image-builder:latest \
    --type qcow2 \
    --config /config.toml \
    --local \
    quay.io/myorg/my-bootc-image:latest
```

### Direct Install to Disk

```bash
sudo podman run --rm --privileged --pid=host --ipc=host \
    -v /var/lib/containers:/var/lib/containers -v /dev:/dev \
    --security-opt label=type:unconfined_t \
    <image> bootc install to-disk /dev/sdX
```

---

## Security Considerations

### FIPS Mode

FIPS requires **two** independent steps — setting the crypto policy alone is not enough.

1. Set the system-wide crypto policy in the Containerfile:

   ```dockerfile
   RUN dnf install -y crypto-policies-scripts && \
       update-crypto-policies --no-reload --set FIPS
   ```

   (`crypto-policies-scripts` is not installed by default on RHEL 10.)

2. Add the `fips=1` kernel argument via a drop-in, e.g. `01-fips.toml`:

   ```toml
   # Enable FIPS
   kargs = ["fips=1"]
   ```

   ```dockerfile
   COPY 01-fips.toml /usr/lib/bootc/kargs.d/
   ```

A FIPS dracut module is built into the base image. For an Anaconda installation, press TAB at the boot menu and add `fips=1` instead. Verify on the booted system:

```bash
cat /proc/sys/crypto/fips_enabled   # expect 1
update-crypto-policies --show       # expect FIPS
```

### Security Hardening and Compliance

Use `oscap-im` with `scap-security-guide` content, which is adjusted and tested for bootable containers:

```dockerfile
RUN dnf install -y openscap-utils scap-security-guide && dnf clean all
RUN oscap-im --profile <profileID> /usr/share/xml/scap/ssg/content/ssg-rhel10-ds.xml
```

Tailoring is supported (change parameters, deselect rules, select extra ones) but **cannot define new rules**:

```dockerfile
COPY tailoring.xml /usr/share/
RUN oscap-im --tailoring-file /usr/share/tailoring.xml \
    --profile stig_customized /usr/share/xml/scap/ssg/content/ssg-rhel10-ds.xml
```

> ⚠️ **Scanning a bootable image while it runs as a container is misleading.** systemd is not running, so services are not running, and `oscap` reports them as correctly configured even though they are not active. Compliance settings are also not enforcing — a later package or prescript can silently alter them. Validate on a booted system.


### Secrets in Images

Never embed runtime secrets (API keys, database passwords) in the image. Use:

- Mounted secrets at runtime
- First-boot provisioning to inject secrets into `/var`
- Cloud provider secret management services

### Cryptographic Sealing (Technology Preview)

Seal bootc images with UEFI Secure Boot keys for verified boot chains.

---

## Troubleshooting: Transient Packages

DNF in RHEL 9.6 and RHEL 10 can install packages transiently by creating a read-write overlay over `/usr`. Add `--transient` to `dnf install`. It works with packages from any DNF repository (including Satellite) and with packages from local directories.

```bash
sudo dnf install --transient <package>
```

For packages that comply with the Linux Filesystem Hierarchy Standard (FHS), a read-write `/usr` directory is sufficient — they shouldn’t be writing to any other directory, except for `/etc` and `/var`, which are mutable in image mode systems.

All transient packages share the same overlay, and it is discarded on the next boot — so this is for packages you need briefly, not for permanent changes. For anything lasting, add the package to the Containerfile and rebuild.

**When it is useful**

- Verifying whether a package update fixes an issue, before you build a new bootc container image using that update
- Adding troubleshooting tools that run as system services

For example, collecting performance data with the Sysstat utilities:

```bash
sudo dnf install --transient sysstat
sudo systemctl start sysstat-collect.timer
# let it collect for a while, then generate reports
sar
```

Or copy the performance data out of `/var/log/sa/` to another system and generate the reports there.

> ⚠️ **Only `/usr` is transient.** Package scriptlets that write to `/etc` or `/var` persist after the overlay is discarded, leaving those paths out of sync with `/usr`. DNF aborts the transaction when a transient install would modify a configured set of drift-prone paths (such as `/etc/pam.d/*`), but the general hazard remains.

---

## Anti-Patterns and Gotchas

### Things That Will Break

1. **Using `rpm-ostree` to install packages at runtime**: Not supported. All packages go in the Containerfile.
2. **Writing application data to `/usr`**: It's read-only at runtime. Use `/var` for mutable data.
3. **Relying on `/var` content being updated**: It's only seeded on first install. Changes in later image versions are ignored for existing systems.
4. **Running `systemctl restart` in Containerfile**: systemd isn't running during builds.
5. **Using `systemctl enable --now`**: The `--now` flag fails without a running systemd.
6. **Creating directories in `/var` with `mkdir`**: Use `systemd-tmpfiles` instead.
7. **Running `firewall-cmd` in Containerfile**: firewalld isn't running. Use XML config files.
8. **Interactive prompts during build**: No TTY available. Defer to first-boot services.
9. **Checking for `/run/systemd/system`**: This directory doesn't exist during builds.
10. **Globally enabling `/usr/lib/bootc/storage`** in `storage.conf`: Explicitly warned against.

### Subtle Issues

- **`/etc` metadata changes block updates**: Changing uid, gid, or xattrs on an `/etc` file counts as a local modification, preventing the image's version from being applied on update.
- **`/etc/passwd` modification is sticky**: Once modified locally, later image changes to it stop applying (see `nss-altfiles` above).
- **`/var/home` content never updates**: Keys or files placed under `/var/home` in a later image version do not reach existing systems.
- **Kickstart + user customization conflict**: `[customizations.user]` and `[customizations.installer.kickstart]` cannot be combined.
- **btrfs limitations**: No custom mount points under `/var` are supported with btrfs rootfs, and there is no support for creating btrfs subvolumes during build time.
- **`anaconda-iso` reformats the first disk** it finds, automatically and without prompting.
- **NetworkManager keyfiles must be mode `600`**, or NetworkManager silently ignores them.
- **dracut grabs the running kernel version** unless you pass the target version explicitly, which errors during a build.
- **Soft reboot is RHEL 10 only** (see the updates section). Where available, it still excludes kernel, driver, and kernel-argument changes, and does not reset `sysctl` settings.
---

## Complete Example: Web Application Image

Putting it all together, here is a production-oriented Containerfile.

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:10.2

LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.description="Production web application server"
LABEL containers.bootc=1
LABEL ostree.bootable=1

# Install packages, clean up in the same layer
RUN dnf -y install httpd mod_ssl mariadb-server vim-minimal && \
    dnf clean all && \
    rm -rf /var/cache/dnf /var/log/dnf*

# Enable services (never use --now)
RUN systemctl enable httpd mariadb

# Relocate web content out of /var into /usr (so it updates with the image)
RUN mv /var/www /usr/share/www && \
    sed -ie 's,/var/www,/usr/share/www,' /etc/httpd/conf/httpd.conf

# Create system user with fixed UID via sysusers
RUN cat > /usr/lib/sysusers.d/webapp.conf << 'EOF'
u webapp 901 "Web Application Service" /var/lib/webapp /sbin/nologin
EOF

# Ensure /var directories exist at boot via tmpfiles
RUN cat > /usr/lib/tmpfiles.d/webapp.conf << 'EOF'
d /var/lib/webapp 0750 webapp webapp -
d /var/log/webapp 0755 webapp webapp -
d /var/lib/mysql 0755 mysql mysql -
d /var/log/mariadb 0755 mysql mysql -
EOF

# Deploy application and configuration
COPY app/ /usr/share/www/html/
COPY httpd-webapp.conf /etc/httpd/conf.d/webapp.conf

# Deploy systemd service for custom application components
COPY webapp.service /usr/lib/systemd/system/webapp.service
RUN systemctl enable webapp

# Firewall configuration (firewalld XML, not firewall-cmd)
COPY webapp-firewall.xml /usr/lib/firewalld/services/webapp.xml

# First-boot setup for host-specific configuration
COPY first-boot-setup.sh /usr/libexec/webapp/first-boot-setup.sh
COPY webapp-firstboot.service /usr/lib/systemd/system/webapp-firstboot.service
RUN systemctl enable webapp-firstboot

# Relabel SELinux contexts
RUN restorecon -R /usr/share/www /usr/lib/systemd/system /usr/libexec/webapp

# Validate the image for bootc compatibility
RUN bootc container lint

# Set CMD for container-mode testing
CMD ["/sbin/init"]
```

---

## Quick Reference

### Key Commands

| Command | Purpose |
|---------|---------|
| `podman build -f Containerfile -t my-bootc-image .` | Build the image |
| `podman run -it --name test -p 8080:80 my-bootc-image` | Test as a container |
| `bootc upgrade --check` | Check whether an update exists — does **not** download |
| `bootc upgrade` | Pull and stage an update |
| `bootc upgrade --apply` | Pull, stage, and reboot |
| `bootc upgrade --soft-reboot=auto --apply` | Pull, stage, soft reboot — **RHEL 10 only** |
| `bootc rollback` | Roll back to previous deployment |
| `bootc switch <new-image-ref>` | Switch to a different image |
| `bootc status` | Show staged, booted, and rollback images |
| `bootc container lint` | Validate image for bootc compatibility |
| `bootc-usr-overlay` | Transient writable overlay on `/usr` for debugging |

`bootc update` is an alias for `bootc upgrade`.

### Kernel Location

The kernel must be at `/usr/lib/modules/$kver/vmlinuz` with the initramfs at `initramfs.img` in the same directory. When regenerating the initramfs in a build, pass the kernel version explicitly — dracut otherwise picks up the *running* kernel and errors:

```dockerfile
RUN set -x; kver=$(cd /usr/lib/modules && echo *); \
    dracut -vf /usr/lib/modules/$kver/initramfs.img $kver
```

### Kernel Arguments

Embed them in the image with a TOML drop-in in `/usr/lib/bootc/kargs.d/`:

```toml
kargs = ["console=ttyS0,115200n8"]
match-architectures = ["x86_64", "aarch64"]
```

At install time use `bootc install ... --karg=<arg>`.

### Architecture Support

x86_64-v2, ARMv8.0-A, ppc64le, s390x. GRUB is the bootloader by default, with the exception of IBM Z.
---

## Sources

- [bootc upstream documentation](https://bootc.dev/bootc/)
- [Red Hat RHEL 9 Image Mode documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/index)
- [Red Hat RHEL 10 Image Mode documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/index)
- [bootc-upgrade(8) man page](https://bootc.dev/bootc/man/bootc-upgrade.8.html)
- [Red Hat Community of Practice Image Mode Demo](https://github.com/redhat-cop/redhat-image-mode-demo)
