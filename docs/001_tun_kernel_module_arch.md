# 1. Building the `tun` Linux Kernel Module Standalone on Arch Linux

Compile and load `tun.ko` out-of-tree without rebuilding the entire kernel.

---

## Prerequisites

Install the kernel headers matching your running kernel and a standard C build environment.

```bash
sudo pacman -S linux-headers base-devel
```

If you run a custom or LTS kernel, replace `linux-headers` with the matching variant (e.g. `linux-lts-headers`).

> **Note:** The headers package installs into `/usr/lib/modules/$(uname -r)/build` — this is the path the Makefile will point to.

```bash
# Verify headers exist
ls /usr/lib/modules/$(uname -r)/build
```

---

## Step 1 — Locate the tun Driver Source

The `tun` driver lives in the kernel source tree under `drivers/net/tun.c`. You need to grab just that file — or the full source if you prefer.

> **Warning:** The source must exactly match your running kernel version — mismatches cause build errors or a broken module.

**Option A — grab only `tun.c` from the Arch Linux kernel package source:**

```bash
# Install ABS helper
sudo pacman -S asp
asp checkout linux
cd linux/trunk
makepkg -so   # downloads and extracts source, skips build
cp src/linux-*/drivers/net/tun.c ~/tun-module/
```

**Option B — download only the file from kernel.org (match your running kernel version):**

```bash
KVER=$(uname -r | grep -oP '^\d+\.\d+\.\d+')
mkdir ~/tun-module && cd ~/tun-module
curl -LO "https://raw.githubusercontent.com/torvalds/linux/v${KVER}/drivers/net/tun.c"
```

---

## Step 2 — Create the Out-of-Tree Makefile

Create a minimal `Makefile` in your working directory. The kernel build system takes care of the rest.

```bash
mkdir -p ~/tun-module && cd ~/tun-module
```

Create a file named `Makefile` with the following content:

```makefile
obj-m += tun.o

KDIR := /usr/lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

> **Note:** The indentation in a Makefile must use **tabs**, not spaces. Most editors auto-convert — double-check with `cat -A Makefile | grep '^\^I'`.

Your working directory should now look like:

```
~/tun-module/
├── tun.c        # driver source
└── Makefile     # build script
```

---

## Step 3 — Build the Module

Run `make` from your working directory. The kernel build system will invoke the compiler with all necessary flags automatically.

```bash
cd ~/tun-module
make
```

On success, several files are produced. The one you care about is `tun.ko`:

```
~/tun-module/
├── tun.ko        # the compiled kernel module ✓
├── tun.o
├── tun.mod.c
├── Module.symvers
└── modules.order
```

If you get errors about missing symbols or structure members, the source version likely does not match your running kernel. Recheck with `uname -r` and re-acquire the matching source file.

```bash
# Inspect the module metadata
modinfo ./tun.ko
```

---

## Step 4 — Load and Verify

Load the freshly built module with `insmod` (direct path) or `modprobe` (after installation).

**Direct load (no install needed):**

```bash
sudo insmod ./tun.ko

# Confirm it's loaded
lsmod | grep tun

# Check kernel ring buffer for errors
sudo dmesg | tail -20
```

To unload:

```bash
sudo rmmod tun
```

> **Warning:** If the system already has a built-in or DKMS-managed `tun` module loaded, unload it first with `sudo modprobe -r tun` before inserting yours.

---

## Step 5 — Persist Across Reboots (Optional)

To make the module loadable by name and survive reboots, install it into the kernel module tree and update the dependency map.

```bash
# Copy to the extra modules directory
sudo cp tun.ko /usr/lib/modules/$(uname -r)/extra/

# Rebuild module dependency map
sudo depmod -a

# Now you can load by name
sudo modprobe tun
```

To auto-load at boot, add the module name to the modules-load config:

```bash
echo 'tun' | sudo tee /etc/modules-load.d/tun.conf
```

> **Note:** Pacman kernel upgrades will **not** automatically rebuild your custom module. You'll need to rebuild and reinstall after each kernel update. Consider DKMS for a fully automated approach on frequently-updated systems.

After a kernel upgrade, rebuild and reinstall:

```bash
cd ~/tun-module
make clean && make
sudo cp tun.ko /usr/lib/modules/$(uname -r)/extra/
sudo depmod -a
```

---

## Summary

| Step | Action |
|------|--------|
| Prerequisites | `sudo pacman -S linux-headers base-devel` |
| Get source | Grab `tun.c` matching your exact kernel version |
| Makefile | Create out-of-tree `obj-m` Makefile |
| Build | `make` → produces `tun.ko` |
| Load | `sudo insmod ./tun.ko` |
| Persist | Copy to `/usr/lib/modules/.../extra/` + `depmod -a` |

The most common pitfall is a **source/kernel version mismatch** — always double-check with `uname -r` before grabbing the source file.


# 2. Fixing the Missing `tun` Module for CheckPoint SNX on Arch Linux

### What modules does SNX need?

SNX requires these kernel modules to function:

| Module | Purpose | Config option |
|--------|---------|---------------|
| `tun` | Virtual TUN/TAP network interface (the critical one) | `CONFIG_TUN` |
| `iptable_nat` | NAT for routing VPN traffic | `CONFIG_IP_NF_NAT` |
| `iptable_filter` | Packet filtering | `CONFIG_IP_NF_FILTER` |
| `nf_conntrack` | Connection tracking (NAT dependency) | `CONFIG_NF_CONNTRACK` |
| `ip_tables` | Base iptables framework | `CONFIG_IP_NF_IPTABLES` |

Check which of these are missing on your system:

```bash
for mod in tun iptable_nat iptable_filter nf_conntrack ip_tables; do
  if modprobe --dry-run $mod 2>/dev/null; then
    echo "OK:      $mod"
  else
    echo "MISSING: $mod"
  fi
done
```

---

### Option 1 — Build `tun` as a Standalone Out-of-Tree Module (fastest)

This is the quickest fix. You build only the `tun` module against your existing kernel's headers, without recompiling the whole kernel.

#### Step 1 — Install kernel headers

```bash
# The headers must exactly match your running kernel version
sudo pacman -S linux-headers

# Verify they match
uname -r
ls /usr/lib/modules/
# Both should show the same version string
```

If you built a custom kernel, you need the headers from that build. If you ran `make modules_install`, the headers are already at `/usr/lib/modules/$(uname -r)/build`. If not, go to your kernel source directory and run:

```bash
sudo make headers_install INSTALL_HDR_PATH=/usr
```

#### Step 2 — Get the `tun` module source

The cleanest approach is to extract it directly from the kernel source tree. Either use the source you already built from, or pull just the relevant file:

```bash
mkdir -p ~/tun-module
cd ~/tun-module

# Copy tun.c from your kernel source (adjust path if needed)
cp /usr/src/linux-*/drivers/net/tun.c .
```

If you no longer have the kernel source, download the matching version from kernel.org:

```bash
KVER=$(uname -r | sed 's/-.*//')   # strip local suffix, e.g. 6.9.3
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-${KVER}.tar.xz
tar --strip-components=4 -xf linux-${KVER}.tar.xz linux-${KVER}/drivers/net/tun.c
```

#### Step 3 — Write the Makefile

```bash
cat > Makefile << 'EOF'
obj-m := tun.o

KDIR := /lib/modules/$(shell uname -r)/build
PWD  := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
EOF
```

#### Step 4 — Build and install

```bash
make

# You should see tun.ko produced
ls -lh tun.ko

# Install it into the module tree
sudo cp tun.ko /lib/modules/$(uname -r)/kernel/drivers/net/tun.ko

# Update module dependency database
sudo depmod -a

# Load it
sudo modprobe tun

# Verify
lsmod | grep tun
```

#### Step 5 — Persist across reboots

```bash
echo "tun" | sudo tee /etc/modules-load.d/tun.conf
```

> **Note:** After every kernel update you will need to rebuild this module. See the tip at the end about automating this with a pacman hook.

---

### Option 2 — Rebuild Your Custom Kernel with `tun` Enabled

If Option 1 fails (e.g. the out-of-tree build errors due to kernel config mismatches), or you want a permanent clean solution, rebuild the kernel with `tun` and all SNX-required options set to `=m`.

#### Step 1 — Go to your kernel source

```bash
cd /usr/src/linux-6.9.3-mycustom   # adjust to your path
```

#### Step 2 — Enable the required options

Run menuconfig and enable each missing module:

```bash
make menuconfig
```

Navigate to and set these to `M` (module):

```
Device Drivers
  → Network device support
    → [M] Universal TUN/TAP device driver        # CONFIG_TUN

Networking support
  → Networking options
    → Network packet filtering framework (Netfilter)
      → IP: Netfilter Configuration
        → [M] IP tables support                  # CONFIG_IP_NF_IPTABLES
        → [M] Packet filtering                   # CONFIG_IP_NF_FILTER
        → [M] IPv4 NAT                           # CONFIG_IP_NF_NAT
      → Core Netfilter Configuration
        → [M] Netfilter connection tracking      # CONFIG_NF_CONNTRACK
```

Or set them directly in `.config` without menuconfig:

```bash
scripts/config --module CONFIG_TUN
scripts/config --module CONFIG_IP_NF_IPTABLES
scripts/config --module CONFIG_IP_NF_FILTER
scripts/config --module CONFIG_IP_NF_NAT
scripts/config --module CONFIG_NF_CONNTRACK

# Resolve any new dependencies
make olddefconfig
```

#### Step 3 — Build only the modules (faster than a full rebuild)

If the kernel image itself doesn't need to change, you can rebuild just the modules:

```bash
make -j$(nproc) modules
```

If you also changed options that affect the kernel image (uncommon for just these networking modules):

```bash
make -j$(nproc)
```

#### Step 4 — Install

```bash
sudo make modules_install

# Only needed if you rebuilt the full kernel image
sudo make install
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

#### Step 5 — Load the new modules

No reboot required if you only rebuilt modules and the kernel version string hasn't changed:

```bash
sudo depmod -a
sudo modprobe tun
sudo modprobe iptable_nat
sudo modprobe iptable_filter
sudo modprobe nf_conntrack

# Persist
cat << 'EOF' | sudo tee /etc/modules-load.d/snx.conf
tun
iptable_nat
iptable_filter
nf_conntrack
EOF
```

If the kernel version string changed, reboot and select the new kernel from GRUB.

---

### Automate module rebuild on kernel updates (pacman hook)

Since Arch updates kernels via pacman, a hook can automatically rebuild your out-of-tree `tun` module after each kernel upgrade:

```bash
sudo mkdir -p /etc/pacman.d/hooks

sudo tee /etc/pacman.d/hooks/tun-module.hook << 'EOF'
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = linux

[Action]
Description = Rebuilding tun.ko for new kernel...
When = PostTransaction
Exec = /usr/local/bin/rebuild-tun.sh
EOF
```

```bash
sudo tee /usr/local/bin/rebuild-tun.sh << 'EOF'
#!/bin/bash
set -e
cd /usr/local/src/tun-module
make clean
make
KVER=$(uname -r)
cp tun.ko /lib/modules/${KVER}/kernel/drivers/net/tun.ko
depmod -a
echo "tun.ko rebuilt and installed for kernel ${KVER}"
EOF

sudo chmod +x /usr/local/bin/rebuild-tun.sh

# Move your tun-module build directory to the permanent location
sudo mv ~/tun-module /usr/local/src/tun-module
```

---

### Verify SNX works after loading modules

```bash
# Load everything
sudo modprobe tun
sudo modprobe iptable_nat

# Check the tun device is available
ls /dev/net/tun

# Try SNX
snx -s vpn.yourcompany.com -c /path/to/cert.p12

# Once connected, confirm the tunnel interface appeared
ip link show | grep tun
```

If SNX still complains after loading `tun`, run it with verbose output and check `dmesg` simultaneously:

```bash
sudo dmesg -w &
snx -s vpn.yourcompany.com -c /path/to/cert.p12
```

---

# 3. Classic SNX issue on Arch. The `tun` module is built into the kernel but may not be loaded

```bash
# Check if tun is loaded
lsmod | grep tun

# Load it manually
sudo modprobe tun

# Verify it loaded
lsmod | grep tun
```

If that works, make it persistent across reboots:

```bash
echo "tun" | sudo tee /etc/modules-load.d/tun.conf
```

If `modprobe tun` fails with "module not found", the module may be missing from your custom kernel. Check if it's compiled in or available:

```bash
# Check if tun is built directly into the kernel (not as a module)
zcat /proc/config.gz | grep CONFIG_TUN
# or
cat /boot/config-$(uname -r) | grep CONFIG_TUN
```

You want to see one of:

- `CONFIG_TUN=m` — built as a module, `modprobe` should work
- `CONFIG_TUN=y` — compiled into the kernel itself, no module needed but SNX may still complain
- `CONFIG_TUN=n` — not compiled at all, you need to rebuild the kernel with it enabled

If it's `=n` and you built a custom kernel (which given this conversation you likely did), go back to your kernel source and enable it:

```
Device Drivers → Network device support → Universal TUN/TAP device driver
```

Set it to `M` (module) rather than `Y`, recompile, and reinstall.