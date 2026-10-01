# Bootc Container Image Best Practices

A comprehensive guide for building bootable container (bootc) images and migrating applications into them. Compiled from the official bootc documentation, Red Hat RHEL 10 Image Mode documentation, and the Red Hat Community of Practice demo repository.

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
| `/opt` | **Read-only** by default (with composefs) | Third-party software expecting to write here needs special handling (see below). |
| `/run`, `/proc`, `/sys` | API filesystems | Shipping content here in images is **not supported**. |
| `/sysroot` | Physical root | The actual host filesystem; deployment roots are subdirectories. |

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

Three approaches, in order of preference:

1. **Symlink specific subdirs into `/var`**: `RUN ln -s /var/opt/myapp /opt/myapp` for maximum immutability
2. **State overlays**: `RUN systemctl enable ostree-state-overlay@opt.service` for a persistent writable overlay
3. **Transient root**: Makes everything writable until next reboot (least recommended)

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

# Declare version
LABEL org.opencontainers.image.version="1.0.0"
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
FROM registry.access.redhat.io/ubi10/go-toolset:latest AS builder
COPY . /opt/app-root/src/
RUN go build -o /opt/app-root/src/myapp

FROM registry.redhat.io/rhel10/rhel-bootc:10.2
COPY --from=builder /opt/app-root/src/myapp /usr/local/bin/myapp
COPY myapp.service /usr/lib/systemd/system/myapp.service
RUN systemctl enable myapp
```

The deployment image should include **only the application and its required runtime**, never build tools.

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

### Interactive Users

For demo/development, users can be created directly:

```dockerfile
RUN dnf -y install mkpasswd
RUN pass=$(mkpasswd --method=SHA-512 --rounds=4096 mypassword) && \
    useradd -m -G wheel myuser -p $pass
RUN echo "%wheel ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/wheel-sudo
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

### Quadlet / Bound Container Images

Applications that run as containers on top of the bootc host can be **logically bound** to the system image:

```dockerfile
# Create a .container quadlet file
COPY myapp.container /usr/share/containers/systemd/myapp.container

# Bind the image to the bootc lifecycle
RUN ln -s /usr/share/containers/systemd/myapp.image \
    /usr/lib/bootc/bound-images.d/myapp.image
```

Logically bound images are pulled/updated when `bootc upgrade` runs and garbage collected when no longer referenced.

> **Warning**: Do not globally enable `/usr/lib/bootc/storage` in `/etc/containers/storage.conf`.

---

## Updates and Rollbacks

### How Updates Work

1. Build a new version of your container image
2. Push to your registry
3. On the deployed system, run `bootc update` (or rely on automatic timer-based updates)
4. bootc fetches **only the changed layers** from the registry
5. A reboot (or soft reboot for non-kernel changes) applies the update

```bash
# Standard update with reboot
sudo bootc update
sudo reboot

# Soft reboot (no kernel change)
sudo bootc update --soft-reboot=required --apply

# Download only, apply later
sudo bootc update --check
```

### Automatic Updates

Enabled by default via systemd timer units. Can be disabled for manual control.

### Rollbacks

bootc keeps the previous deployment. Boot into it via the bootloader menu, or:

```bash
sudo bootc rollback
sudo reboot
```

### Image Switching

Switch to an entirely different image:

```bash
sudo bootc switch quay.io/myorg/new-image:latest
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
| `anaconda-iso` | Unattended Anaconda installer |
| `vhd` | Azure / Virtual PC |
| `gce` | Google Compute Engine |

### Requirements

- **Rootful Podman required**: rootless mode causes build failure
- Must run with `--privileged` and SELinux unconfined
- Cannot pull images from remote registries directly; images must be in local storage

```bash
sudo podman run --rm --privileged \
    --security-opt label=type:unconfined_t \
    -v /var/lib/containers/storage:/var/lib/containers/storage \
    quay.io/centos-bootc/bootc-image-builder:latest \
    --type qcow2 \
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

### Converting an Existing System

```bash
sudo podman run --rm --privileged -v /dev:/dev \
    -v /var/lib/containers:/var/lib/containers -v /:/target \
    --pid=host --security-opt label=type:unconfined_t \
    <image> bootc install to-existing-root
```

Previous system data remains accessible at `/sysroot` after reboot.

---

## Security Considerations

### Composefs and Integrity

Enable composefs for a fully read-only root filesystem:

```ini
# /usr/lib/ostree/prepare-root.conf
[composefs]
enabled = true
```

For hard-required fsverity (errors if filesystem doesn't support it):

```ini
[composefs]
enabled = verity
```

### SELinux

- Unknown toplevel directories may get `default_t` label, making them inaccessible
- After build, verify contexts: `RUN restorecon -R <path>`
- Custom file contexts may need explicit label definitions

### FIPS Mode

Can be enabled during bootc image build for compliance requirements.

### Private Registries

- Embed pull secrets at `/etc/ostree/auth.json` in the image
- Or inject via kickstart / cloud-init at provisioning time
- TLS can be disabled for testing (never in production)

### Secrets in Images

Never embed runtime secrets (API keys, database passwords) in the image. Use:

- Mounted secrets at runtime
- First-boot provisioning to inject secrets into `/var`
- Cloud provider secret management services

### Cryptographic Sealing (Technology Preview)

Seal bootc images with UEFI Secure Boot keys for verified boot chains.

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
- **ostree squashes all timestamps to zero**: This is a known implementation bug that will be fixed in the future.
- **Kickstart + user customization conflict**: `[customizations.user]` and `[customizations.installer.kickstart]` cannot be combined.
- **btrfs limitations**: No custom mount points under `/var` are supported with btrfs rootfs, and there is no support for creating btrfs subvolumes during build time.
- **Soft reboot limitations**: Only works for non-kernel changes. Kernel or initramfs updates require a full reboot.
- **`systemd-confext` and `systemd-sysext` are not supported** on bootc-managed systems due to overlayfs conflicts.

---

## Complete Example: Web Application Image

Putting it all together, here is a production-oriented Containerfile:

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:10.2

LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.description="Production web application server"

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

# Fix SELinux contexts
RUN restorecon -R /usr/share/www /usr/lib/systemd/system /usr/libexec/webapp

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
| `bootc install to-disk /dev/sdX` | Install to bare disk |
| `bootc install to-existing-root` | Convert running system |
| `bootc update` | Pull and stage an update |
| `bootc update --apply` | Pull, stage, and reboot |
| `bootc rollback` | Roll back to previous deployment |
| `bootc switch <new-image-ref>` | Switch to a different image |
| `bootc status` | Show current deployment status |
| `bootc container lint` | Validate image for bootc compatibility |

### Kernel Location

The kernel must be at `/usr/lib/modules/$kver/vmlinuz` with the initramfs at `initramfs.img` in the same directory.

### Architecture Support

x86_64-v2, ARMv8.0-A, ppc64le, s390x (with caveats: Anaconda doesn't work on s390x and ppc64le).

---

## Sources

- [bootc upstream documentation](https://bootc.dev/bootc/)
- [Red Hat RHEL 10 Image Mode documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/index)
- [Red Hat Community of Practice Image Mode Demo](https://github.com/redhat-cop/redhat-image-mode-demo)
