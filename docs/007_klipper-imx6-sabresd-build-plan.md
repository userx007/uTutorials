# Klipper Host on NXP i.MX6Q SABRE-SD + Mellow FLY D7 — Build Plan

**Key facts confirmed via search:**
- The MCIMX6Q-SDB is supported by **meta-freescale** (Yocto) under `MACHINE = imx6qdlsabresd` — this covers the i.MX6Q/DL/QP SABRE-SD variants.On master, imx6qdlsabresd.conf supports all mx6sabresd variants: imx6q, imx6dl, imx6qp
- FLY D7 specs: 32-bit ARM Cortex-M0+ STM32F072RBT6 at 48MHz, with CAN bus support, and Klipper firmware requires a companion computer (host). It does not support reverse-powering the host through its interfaces — so the SABRE board needs its own power supply, not power pulled off the D7.
- Firmware flashing uses either USB (Katapult bootloader route) or the CAN-USB bridge method Mellow documents.

- **Yocto over Buildroot** for this specific pairing — meta-freescale already has a maintained `imx6qdlsabresd` machine and prior art for LVDS bring-up on this exact board family, which saves you from writing device-tree/panel timing work from scratch.
- **Power topology**: the FLY D7 explicitly can't back-power a host, so the SABRE-SD needs its own supply — don't wire it expecting USB bus power from the D7.
- **Klipper has no official Yocto recipe** — you'll install it as a systemd-managed app on the built rootfs (or write your own `.bb` once your config stabilizes).
- **Bootloader offset**: match Klipper's `make menuconfig` STM32 bootloader offset to whatever Mellow's Katapult bootloader on the D7 expects, or you'll get a board that flashes successfully but won't boot Klipper.

## 1. System architecture

```
[MC IMX-LVDS-1 panel] --LVDS--> [MCIMX6Q-SDB SABRE-SD]  <-- runs: Linux + Klipper (klippy) + Moonraker + KlipperScreen
                                        |
                                USB (or USB-CAN bridge)
                                        |
                              [Mellow FLY D7 mainboard]  <-- runs: Klipper MCU firmware (STM32F072RBT6)
                                        |
                          steppers / heaters / thermistors / endstops / BLTouch
```

Two separate firmware builds are involved:
1. **Host side**: embedded Linux distro for the i.MX6Q, running `klippy` (the Python Klipper host process), Moonraker (API server), and KlipperScreen (touch UI on the LVDS panel).
2. **MCU side**: the actual Klipper firmware compiled for the FLY D7's STM32F072RBT6, flashed onto the board itself.

Important constraint from the FLY D7 docs: **the board does not back-power a host computer** over USB/UART — the SABRE-SD needs its own 5V supply (its DC barrel/PMIC input), it can't be parasitically powered from the D7 like some small SBCs are.

---

## 2. Choosing Yocto vs Buildroot for this board

**Recommendation: Yocto**, specifically the `meta-freescale` BSP layer, not Buildroot. Reasons specific to your hardware:

- The MCIMX6Q-SDB has a maintained Yocto machine definition (`imx6qdlsabresd`) directly from the meta-freescale project, covering i.MX6Q/DL/QP variants of the SABRE-SD.
- LVDS display bring-up on this board (IPU/LDB driver, device-tree overlays for panel timings) is a known, previously-solved path in meta-freescale/NXP's kernel branches — there's existing community troubleshooting for exactly this panel class (10" LVDS on SABRE-SD), which is worth searching if you hit a blank-screen issue.
- NXP's own BSP (`fsl-community-bsp` / `fsl-arm-yocto-bsp`) already targets `imx6qsabresd` and gives you a known-good kernel/u-boot baseline you can layer Klipper on top of, rather than writing DTS/board bring-up from scratch as Buildroot would require.
- Buildroot *can* target i.MX6 boards too, and produces smaller/faster builds, but you'd be hand-rolling the LVDS panel device tree and IPU driver config with much less prior art to draw on for this exact board+panel combo. If your priority is minimal image size and boot time, and you're comfortable doing the display bring-up yourself, Buildroot is viable — but Yocto is the lower-risk path here.

---

## 3. Host build: Yocto with meta-freescale

### 3.1 Build host prerequisites
- Ubuntu 22.04/24.04 x86_64 build machine (Yocto builds are heavy: 100GB+ disk, 8+ GB RAM recommended).
- Install standard Yocto host packages (`gawk wget git diffstat unzip texinfo build-essential chrpath socat cpio python3 python3-pip python3-pexpect xz-utils debianutils iputils-ping libsdl1.2-dev xterm` etc. — exact list depends on the Yocto release's `docs/ref-manual` for your chosen branch).

### 3.2 Fetch layers
```bash
mkdir ~/imx6-klipper && cd ~/imx6-klipper
git clone -b <release-branch> git://git.yoctoproject.org/poky
git clone -b <release-branch> https://github.com/Freescale/meta-freescale
git clone -b <release-branch> https://github.com/Freescale/meta-freescale-3rdparty
git clone -b <release-branch> https://git.openembedded.org/meta-openembedded
```
Use a matching `<release-branch>` (e.g. a recent LTS Yocto release name) across all layers — meta-freescale tracks poky release names, so pick whichever current LTS meta-freescale supports and use that same branch/tag everywhere.

### 3.3 Initialize the build
```bash
source poky/oe-init-build-env build-sabresd
bitbake-layers add-layer ../meta-freescale
bitbake-layers add-layer ../meta-freescale-3rdparty
bitbake-layers add-layer ../meta-openembedded/meta-oe
bitbake-layers add-layer ../meta-openembedded/meta-python
bitbake-layers add-layer ../meta-openembedded/meta-networking
```

### 3.4 conf/local.conf
```
MACHINE = "imx6qdlsabresd"
ACCEPT_FSL_EULA = "1"
DISTRO ?= "fsl-imx-x11"      # or fsl-imx-wayland if you want Wayland instead of X11
IMAGE_INSTALL:append = " python3 python3-pip python3-cffi python3-numpy \
  python3-jinja2 python3-pyserial python3-greenlet libffi \
  git dbus avahi-daemon openssh nano \
  packagegroup-core-x11-base packagegroup-core-x11-sato"
```
- `fsl-imx-x11` is the simplest path for getting KlipperScreen (a GTK app) running on the LVDS panel. Wayland is possible but adds complexity for a GTK-based UI.
- Klipper itself has no Yocto recipe upstream — you'll add it as a `git`/`systemd`-managed application rather than a proper bitbake recipe (see §5), or you can write a minimal recipe if you want it baked into the rootfs image itself.

### 3.5 LVDS panel device tree
The MC IMX-LVDS-1 needs its panel timings (resolution, clock, polarity) matched in the kernel device tree (`imx6q-sabresd.dts` or an overlay) under the `ldb`/`lvds-channel` nodes, and the correct `ipu1_csi`/`ldb` bootargs (`video=mxcfb0:dev=ldb...`). NXP's SABRE-SD Linux kernel branch already has LVDS support scaffolded — you mainly need to set the panel's native timings (check the MC IMX-LVDS-1 datasheet for pixel clock, h/v sync, porch values) and confirm which LVDS channel (`ldb=dul0` vs `dul1` vs split-mode) matches your panel's wiring. If you get a blank screen after boot with X server running (a known reported symptom on this board class), try the other LVDS channel first — it's the most common cause.

### 3.6 Build and flash
```bash
bitbake core-image-x11
# result: build-sabresd/tmp/deploy/images/imx6qdlsabresd/*.sdcard
sudo dd if=core-image-x11-imx6qdlsabresd.sdcard of=/dev/sdX bs=1M status=progress conv=fsync
```
Boot the SABRE-SD from SD card, confirm console + network + (once panel DT is right) the LVDS display comes up.

---

## 4. MCU firmware: building Klipper for the FLY D7

This part happens **on the host** (SABRE-SD or any Linux PC — the compiled binary is what matters, not where you compile it) using the standard Klipper firmware build flow, since Mellow doesn't ship pre-built Klipper binaries for arbitrary configs.

```bash
git clone https://github.com/Klipper3d/klipper
cd klipper
make menuconfig
```
In `menuconfig`, select:
- Micro-controller Architecture: **STMicroelectronics STM32**
- Processor model: **STM32F072**
- Bootloader offset: depends on flashing method —
  - **8 KiB bootloader offset** if using Mellow's Katapult/CAN-bridge flashing method (Mellow's docs describe flashing via USB-CAN bridge with the LED-blink recovery mode).
  - **No bootloader / direct USB DFU** if flashing via STM32 built-in USB DFU instead.
- Communication interface: **USB** (simplest for direct host connection) or **CAN bus** (recommended if you want to run the SABRE-SD physically separated from the printer electronics — needs a USB-CAN adapter or the board's onboard CAN transceiver wired to a CAN interface on the host side).

```bash
make
# produces out/klipper.bin
```

Flashing (per Mellow's documented flow):
1. Connect FLY D7 to the host via USB-C, double-tap reset to enter the bootloader (LED blinks).
2. Confirm the board enumerates: `lsusb` should show device ID `1d50:6177` (Katapult bootloader).
3. Use `flashtool.py` / `flash_can.py` from the Katapult repo (Mellow's boards ship Katapult, not the legacy Klipper DFU bootloader) to write `klipper.bin`.
4. For CAN-based flashing after the first USB flash, subsequent updates can go over CAN using the same Katapult tooling.

---

## 5. Klipper host-side install (klippy + Moonraker)

Do this on the SABRE-SD once it boots to Linux:

```bash
git clone https://github.com/Klipper3d/klipper
cd klipper
python3 -m venv ~/klippy-env
~/klippy-env/bin/pip install -r scripts/klippy-requirements.txt
```
Set up `printer.cfg` with an `[mcu]` section pointing at the FLY D7's serial device or CAN UUID:
```ini
[mcu]
serial: /dev/serial/by-id/usb-Klipper_stm32f072...   # USB mode
# or, for CAN:
# canbus_uuid: <uuid from `python3 canbus_query.py`>
```
Run Klipper as a systemd service (`klipper.service`) pointing at `klippy.py`, `printer.cfg`, and the API socket.

Install **Moonraker** (API layer) similarly via its install script or manually cloning + venv, configured to talk to Klipper's Unix domain socket.

Install a frontend: **Mainsail** or **Fluidd** (web UI, served via nginx — add `nginx` to your Yocto image if you want local web-UI access) and/or **KlipperScreen** for the LVDS touch panel — KlipperScreen is a Python/GTK3 app, so make sure `python3-gobject`, `gtk3`, and related packages are in `IMAGE_INSTALL` for your Yocto build (§3.4).

---

## 6. Bring-up / test sequence

1. Boot SABRE-SD standalone, confirm console + network + LVDS display + touch (if the panel has touch) all work *before* touching Klipper.
2. Flash and verify Klipper MCU firmware on the FLY D7 in isolation (bench test, no motors wired) — confirm `klippy` connects and `STATUS` reports "Ready" via a minimal `printer.cfg`.
3. Wire one axis + one thermistor + one heater, verify manual moves (`G28`, `G1`) and PID/temp reporting.
4. Wire remaining axes/extruder/CAN or USB link, full `printer.cfg` with your kinematics (CoreXY/Cartesian/etc. per your printer).
5. Bring up Moonraker + Mainsail/Fluidd, confirm remote control.
6. Bring up KlipperScreen on the LVDS panel, point it at Moonraker's local API.
7. Full dry run (no filament, PLA temps) before printing.

---

## 7. Things likely to bite you

- **LVDS panel timing mismatch** is the most common source of "X11 runs but screen is blank" on this exact board family — verify against the MC IMX-LVDS-1 datasheet's exact pixel clock and porch values rather than reusing another panel's DTS entry.
- **FLY D7 can't power the host** — plan separate power for the SABRE-SD.
- **No official Yocto recipe for Klipper** — you're either scripting it as a post-install/systemd unit on top of the image, or writing your own `.bb` recipe if you want it truly baked into the rootfs (recommended once your config is stable, so updates survive re-flashing).
- **Bootloader offset mismatch** between what you select in Klipper's `make menuconfig` and Mellow's shipped Katapult bootloader will produce a board that flashes but doesn't run — match it to Mellow's documented offset for the D7, not a generic STM32F072 default.
