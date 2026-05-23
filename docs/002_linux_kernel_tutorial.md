# Building and Installing a Custom Linux Kernel

A complete guide to compiling a modified kernel and installing it across major Linux distributions.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Getting the Kernel Source](#getting-the-kernel-source)
4. [Configuring the Kernel](#configuring-the-kernel)
5. [Compiling the Kernel](#compiling-the-kernel)
6. [Installing the Kernel](#installing-the-kernel)
   - [Debian / Ubuntu](#debian--ubuntu)
   - [Fedora / RHEL / CentOS](#fedora--rhel--centos)
   - [Arch Linux / Manjaro](#arch-linux--manjaro)
   - [openSUSE](#opensuse)
   - [Gentoo](#gentoo)
7. [Configuring the Bootloader](#configuring-the-bootloader)
8. [Verifying the New Kernel](#verifying-the-new-kernel)
9. [Rolling Back](#rolling-back)
10. [Tips and Best Practices](#tips-and-best-practices)

---

## Overview

Building a custom kernel lets you tailor Linux to your exact hardware and workload — stripping out unused drivers, enabling experimental features, applying custom patches, or hardening the system for security. The general workflow is the same across all distributions:

1. Install build dependencies
2. Download the kernel source
3. Configure the kernel options
4. Compile
5. Install the new kernel and its modules
6. Update the bootloader

Distribution-specific steps are covered in the [Installing the Kernel](#installing-the-kernel) section.

> **Warning:** A misconfigured kernel can prevent your system from booting. Always keep a known-good kernel available and test in a virtual machine first when possible.

---

## Prerequisites

### Build Dependencies

Install the required tools before anything else.

**Debian / Ubuntu:**
```bash
sudo apt update
sudo apt install -y build-essential libncurses-dev bison flex libssl-dev \
  libelf-dev dwarves zstd bc wget
```

**Fedora / RHEL / CentOS:**
```bash
sudo dnf groupinstall "Development Tools"
sudo dnf install -y ncurses-devel bison flex openssl-devel elfutils-libelf-devel \
  dwarves zstd bc wget
```

**Arch Linux:**
```bash
sudo pacman -S base-devel ncurses bison flex openssl libelf pahole zstd bc wget
```

**openSUSE:**
```bash
sudo zypper install -t pattern devel_basis
sudo zypper install ncurses-devel bison flex libopenssl-devel libelf-devel \
  dwarves zstd bc wget
```

**Gentoo:**
```bash
sudo emerge --ask sys-devel/bc dev-libs/openssl dev-util/pahole app-arch/zstd
```

### Disk Space and RAM

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| Disk space | 15 GB | 30 GB |
| RAM | 2 GB | 8 GB |
| CPU cores | 1 | 4+ (parallel compilation) |

---

## Getting the Kernel Source

### Option A — Download from kernel.org (recommended)

```bash
# Check the latest stable version at https://kernel.org
KERNEL_VERSION=6.9.3   # replace with desired version

cd /usr/src
sudo wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-${KERNEL_VERSION}.tar.xz
sudo wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-${KERNEL_VERSION}.tar.sign

# Verify the signature (optional but recommended)
sudo xz -d linux-${KERNEL_VERSION}.tar.xz
gpg --verify linux-${KERNEL_VERSION}.tar.sign

# Extract
sudo tar -xf linux-${KERNEL_VERSION}.tar
cd linux-${KERNEL_VERSION}
```

### Option B — Clone from Git

```bash
git clone --depth=1 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
```

For a specific stable branch:

```bash
git clone --depth=1 --branch linux-6.9.y \
  https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
```

### Option C — Use Your Distribution's Source Package

This approach starts from the config already used by your distro, which is the safest baseline.

**Debian / Ubuntu:**
```bash
sudo apt install linux-source
cd /usr/src
sudo tar -xf linux-source-*.tar.bz2
cd linux-source-*/
```

**Fedora:**
```bash
sudo dnf install kernel-devel
# Source RPM:
dnf download --source kernel
rpm -ivh kernel-*.src.rpm
```

**Arch Linux**

```bash
# Install asp (Arch Source Package tool)
sudo pacman -S asp

# Fetch the official linux package build files
asp export linux
cd linux/

# The PKGBUILD, config, and patches are now local
ls
```

Then pull the actual kernel source tarball referenced in the PKGBUILD:

```bash
makepkg --nobuild   # downloads and extracts sources without building
# Source lands in src/linux-<version>/
cd src/linux-*/
```

From there you have the full kernel source tree with Arch's config already in place, and you can proceed with `make menuconfig` as usual.

---

## Configuring the Kernel

All configuration is stored in a file called `.config` in the kernel source directory. There are several ways to create or modify it.

### Start from Your Running Kernel's Config

This is the safest starting point — it copies the config that your distribution already uses and tested:

```bash
# Method 1: copy from /boot
cp /boot/config-$(uname -r) .config

# Method 2: read from the running kernel (if CONFIG_IKCONFIG is set)
zcat /proc/config.gz > .config

# Adjust the config to match any new kernel options
make olddefconfig
```

### Interactive Configuration Tools

Choose one of these interfaces to review and change options:

```bash
# Terminal menu (most common)
make menuconfig

# Qt-based GUI (requires Qt dev libraries)
make xconfig

# GTK-based GUI
make gconfig

# Simple text-based, question-by-question
make config
```

### Common Configuration Changes

Navigate the menu using arrow keys. Press `Y` to build a feature into the kernel, `M` to build it as a loadable module, or `N` to exclude it. Press `/` to search for an option by name.

**Disable debugging symbols** (reduces size significantly):
```
Kernel hacking → Compile-time checks and compiler options
  → Debug information → None
```

**Enable processor-specific optimizations:**
```
Processor type and features → Processor family
  → (select your CPU, e.g., Core 2/newer Xeon)
```

**Set a custom kernel version string** (helps you identify your build):
```
General setup → Local version - append to kernel release
  → "-mycustom"
```

**Enable Realtime scheduling (PREEMPT_RT patch):**
```
General setup → Preemption Model
  → Fully Preemptible Kernel (Real-Time)
```

### Applying Patches

Apply patches before running `make menuconfig`:

```bash
# Single patch
patch -p1 < my-patch.patch

# Series of patches
for p in patches/*.patch; do
  patch -p1 < "$p" && echo "Applied: $p"
done
```

### Reduce Compilation Time: Strip Unused Modules

If starting from a distribution config, many modules are enabled that your hardware may not need. This script disables modules not currently loaded:

```bash
# Run as root while all needed hardware is active
make localmodconfig
```

> **Note:** Only run this on the machine you are building for. Do not use it when cross-compiling.

---

## Compiling the Kernel

### Build

Use `-j` to parallelize across CPU cores. A good rule of thumb is `$(nproc)` for all available cores, or `$(nproc) + 1`:

```bash
make -j$(nproc)
```

Compilation typically takes 15–90 minutes depending on your hardware and how many features are enabled. You can watch progress with:

```bash
make -j$(nproc) 2>&1 | tee build.log
```

### Build Only Modules

```bash
make -j$(nproc) modules
```

### Cross-Compilation (optional)

To build for a different architecture, set `ARCH` and `CROSS_COMPILE`:

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

---

## Installing the Kernel

The installation steps differ by distribution because of package management and bootloader conventions.

---

### Debian / Ubuntu

Debian and Ubuntu recommend building a `.deb` package so the kernel is tracked by `apt` and can be removed cleanly.

#### Method 1 — Build a .deb Package (recommended)

```bash
# From the kernel source directory
make -j$(nproc) bindeb-pkg LOCALVERSION=-mycustom

# The packages appear one directory above
ls ../linux-*.deb
```

This produces packages such as:

- `linux-image-6.9.3-mycustom_amd64.deb` — the kernel and modules
- `linux-headers-6.9.3-mycustom_amd64.deb` — header files
- `linux-libc-dev_*.deb` — userspace development headers

Install them:

```bash
sudo dpkg -i ../linux-image-*.deb ../linux-headers-*.deb
```

GRUB is updated automatically via the `postinst` script.

#### Method 2 — Manual Install

```bash
sudo make modules_install
sudo make install
sudo update-grub
```

#### Removal

```bash
sudo apt remove linux-image-6.9.3-mycustom
sudo update-grub
```

---

### Fedora / RHEL / CentOS

Fedora and its relatives use RPM packages and manage kernels with `grubby`.

#### Method 1 — Build an RPM Package (recommended)

```bash
make -j$(nproc) rpm-pkg LOCALVERSION=-mycustom

# RPMs appear in ~/rpmbuild/RPMS/
ls ~/rpmbuild/RPMS/x86_64/
```

Install:

```bash
sudo rpm -ivh ~/rpmbuild/RPMS/x86_64/kernel-*.rpm
```

Or use DNF:

```bash
sudo dnf install ~/rpmbuild/RPMS/x86_64/kernel-6.9.3-mycustom.x86_64.rpm
```

The new kernel is added to the GRUB menu automatically.

#### Method 2 — Manual Install

```bash
sudo make modules_install
sudo make install

# Verify it was added
sudo grubby --info=ALL | grep title
```

#### Set the Default Kernel

```bash
sudo grubby --set-default /boot/vmlinuz-6.9.3-mycustom
```

#### Removal

```bash
sudo rpm -e kernel-6.9.3-mycustom
```

---

### Arch Linux / Manjaro

Arch installs kernels directly without packaging them — but you can also create a proper PKGBUILD for cleaner management.

#### Method 1 — Direct Install

```bash
sudo make modules_install

# Copy kernel image, System.map, and config
sudo cp arch/x86/boot/bzImage /boot/vmlinuz-mycustom
sudo cp System.map /boot/System.map-mycustom
sudo cp .config /boot/config-mycustom

# Generate an initramfs
sudo mkinitcpio -k 6.9.3-mycustom -g /boot/initramfs-mycustom.img
```

Update GRUB:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

#### Method 2 — PKGBUILD (recommended for Arch)

The AUR package `linux-custom` provides a template. Alternatively, copy the official `linux` PKGBUILD and modify it:

```bash
# Clone the official linux package as a starting point
asp export linux
cd linux/
# Edit PKGBUILD: change pkgver, add your patches to source[] array
makepkg -si
```

This creates a proper package that `pacman` can track and remove cleanly.

#### Removal

```bash
sudo pacman -R linux-mycustom   # if packaged
# or manually:
sudo rm /boot/vmlinuz-mycustom /boot/initramfs-mycustom.img
sudo rm -r /lib/modules/6.9.3-mycustom
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

### openSUSE

openSUSE uses RPM packages. The workflow is similar to Fedora.

#### Build an RPM

```bash
make -j$(nproc) rpm-pkg LOCALVERSION=-mycustom
sudo zypper install --allow-unsigned-rpm ~/rpmbuild/RPMS/x86_64/kernel-*.rpm
```

openSUSE uses `grub2-mkconfig` and `update-bootloader`. Both run automatically after RPM installation.

#### Manual Install

```bash
sudo make modules_install
sudo make install
sudo update-bootloader --refresh
```

#### Set Default Kernel

```bash
sudo grub2-set-default 0   # 0 = first entry (most recent)
```

#### Removal

```bash
sudo zypper remove kernel-6.9.3-mycustom
sudo update-bootloader --refresh
```

---

### Gentoo

On Gentoo, kernel management is handled manually or via the `kernel-install` tool. Gentoo users often manage kernels through `sys-kernel/gentoo-sources` or a custom overlay.

#### Install via eselect-kernel

```bash
# Copy your source to /usr/src/linux-6.9.3-mycustom
sudo cp -r /path/to/linux-6.9.3 /usr/src/linux-6.9.3-mycustom

# Register and activate it
sudo eselect kernel list
sudo eselect kernel set linux-6.9.3-mycustom

# Verify the symlink
ls -la /usr/src/linux
```

#### Configure and Build

```bash
cd /usr/src/linux

# Copy existing config
sudo cp /usr/src/linux-$(uname -r)/.config .
sudo make olddefconfig
sudo make menuconfig

sudo make -j$(nproc)
sudo make modules_install
```

#### Install with kernel-install (Gentoo ≥ 17.1)

```bash
sudo make install   # triggers /sbin/installkernel or kernel-install
```

#### Manual Install (older systems)

```bash
sudo cp arch/x86/boot/bzImage /boot/kernel-6.9.3-mycustom
sudo cp System.map /boot/System.map-6.9.3-mycustom

# Generate initramfs with dracut or genkernel
sudo dracut /boot/initramfs-6.9.3-mycustom.img 6.9.3-mycustom
# or
sudo genkernel --install initramfs
```

Update GRUB:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

#### Rebuild External Modules

After installing a new kernel on Gentoo, rebuild any external kernel modules:

```bash
sudo emerge --ask @module-rebuild
```

---

## Configuring the Bootloader

Most distribution-specific installation methods update GRUB automatically. If they do not, or if you need to adjust settings, use these commands.

### GRUB 2 (most distributions)

```bash
# Regenerate grub.cfg
sudo grub-mkconfig -o /boot/grub/grub.cfg      # Arch, Gentoo, some others
sudo update-grub                                 # Debian/Ubuntu alias
sudo grub2-mkconfig -o /boot/grub2/grub.cfg     # Fedora/RHEL (BIOS)
sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg  # Fedora/RHEL (UEFI)
```

### Set Default Boot Entry

```bash
# By index (0 = first entry)
sudo grub-set-default 0

# By exact title string
sudo grub-set-default "Advanced options for Ubuntu>Ubuntu, with Linux 6.9.3-mycustom"

# Fedora/RHEL — by path to vmlinuz
sudo grubby --set-default /boot/vmlinuz-6.9.3-mycustom
```

### Timeout

Edit `/etc/default/grub`:

```
GRUB_TIMEOUT=10          # Show menu for 10 seconds
GRUB_TIMEOUT_STYLE=menu  # Always show the menu
```

Then regenerate:

```bash
sudo update-grub   # or grub-mkconfig
```

### systemd-boot (alternative to GRUB)

If your system uses `systemd-boot` (common on modern Arch installs), create a loader entry manually:

```
# /boot/loader/entries/mycustom.conf
title   Linux 6.9.3 (mycustom)
linux   /vmlinuz-mycustom
initrd  /initramfs-mycustom.img
options root=UUID=<your-root-uuid> rw quiet
```

Find your root UUID with:

```bash
blkid /dev/sda2   # replace with your root partition
```

---

## Verifying the New Kernel

After rebooting into your new kernel:

```bash
# Confirm the running kernel version
uname -r
# Expected output: 6.9.3-mycustom

# Verify kernel command-line parameters
cat /proc/cmdline

# Check that modules loaded correctly
lsmod

# Review boot messages for errors
dmesg | grep -i error
dmesg | grep -i warn

# Confirm your hardware is recognized
lspci -k    # PCI devices and their drivers
lsusb       # USB devices
```

---

## Rolling Back

Always keep at least one previous kernel installed. GRUB shows all installed kernels in its menu, so rolling back is as simple as selecting an older entry at boot.

### Debian / Ubuntu — Set Previous Kernel as Default

```bash
# List installed kernels
dpkg --list | grep linux-image

# Boot into the previous kernel once (temporary)
# → Select it manually in the GRUB menu at startup

# Make it permanent
sudo grub-set-default "Advanced options for Ubuntu>Ubuntu, with Linux 5.15.0-91-generic"
sudo update-grub
```

### Fedora / RHEL — Roll Back with grubby

```bash
# List all kernels
sudo grubby --info=ALL

# Set an older kernel as default
sudo grubby --set-default /boot/vmlinuz-5.14.0-362.24.1.el9_3.x86_64
```

### Remove the New Kernel

If the new kernel is broken, boot into an older one and remove the new one:

```bash
# Debian/Ubuntu
sudo apt remove linux-image-6.9.3-mycustom

# Fedora/RHEL
sudo rpm -e kernel-6.9.3-mycustom

# Arch (manual install)
sudo rm /boot/vmlinuz-mycustom /boot/initramfs-mycustom.img
sudo rm -r /lib/modules/6.9.3-mycustom
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

## Tips and Best Practices

### Keep Kernels Manageable

- Name your builds descriptively: `-j$(nproc)` and `LOCALVERSION=-6.9.3-hwaccel-test` makes it clear what you changed.
- Keep no more than 2–3 kernels installed. Old ones consume space in `/boot` and can fill it on small partitions.
- Test new kernels in a VM (QEMU/KVM, VirtualBox) before deploying on real hardware.

### Speed Up Recompilation

After changing only a few config options, incremental compilation is much faster:

```bash
make -j$(nproc)   # only rebuilds what changed
```

Use `ccache` to cache compiled objects across builds:

```bash
sudo apt install ccache   # or dnf/pacman equivalent
export CC="ccache gcc"
make -j$(nproc) CC="ccache gcc"
```

### Kernel Module Signing

If your system uses Secure Boot, unsigned modules are rejected. Sign your modules after compilation:

```bash
# Generate a key pair (once)
openssl req -new -x509 -newkey rsa:2048 -keyout signing_key.pem \
  -out signing_cert.pem -days 365 -subj "/CN=Kernel Module Signing/"

# Sign a module
/usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 \
  signing_key.pem signing_cert.pem my_module.ko
```

Enroll the certificate in the firmware's MOK database with `mokutil`:

```bash
sudo mokutil --import signing_cert.pem
```

### Useful Kernel Parameters for Testing

Add these to your bootloader entry to help debug a new kernel:

| Parameter | Effect |
|-----------|--------|
| `nomodeset` | Disable kernel mode-setting; fallback to VESA |
| `debug` | Verbose kernel output during boot |
| `initcall_debug` | Log every driver initialization call |
| `panic=10` | Auto-reboot 10 seconds after a kernel panic |
| `systemd.unit=rescue.target` | Boot to rescue shell |
| `loglevel=7` | Maximum verbosity in `dmesg` |

Add them temporarily at the GRUB menu by pressing `e` on the kernel entry and appending to the `linux` line.

### Keeping Up with Security Patches

Subscribe to the kernel mailing list or watch [kernel.org](https://www.kernel.org) for stable updates. The stable team backports important fixes into `linux-x.y.z` branches. Apply updates to your custom kernel by pulling the new source and diffing configs:

```bash
# From your kernel source directory
git fetch --tags
git checkout v6.9.4
make olddefconfig   # integrate new options into your existing .config
make -j$(nproc)
```

---

*Last updated for kernel 6.9.x. Commands and package names may vary slightly across distribution versions.*
