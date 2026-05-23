# Building the `tun` Linux Kernel Module Standalone on Arch Linux

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
