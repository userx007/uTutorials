# Klipper Host on NXP i.MX53 Quick Start Board (QSB) + Mellow FLY D7 — Build Plan

## 1. How this differs from the SABRE-SD plan

Same overall architecture (Linux host running klippy/Moonraker talks to the FLY D7 MCU over USB or CAN), but two things change with the i.MX53 QSB:

- **Much weaker SoC**: single-core Cortex-A8 @ up to ~1GHz, typically 512MB RAM, vs the SABRE-SD's quad-core Cortex-A9. Klipper's host process itself is lightweight (people run it on a Pi Zero), so it will work — but don't expect to also run a heavy desktop environment, Wayland compositor, or webcam streaming (mjpg-streamer/crowsnest) comfortably on the same core. Keep the image minimal (console/Sato, not a full X11 desktop with extras) if you're also running Moonraker + a web UI + KlipperScreen concurrently.
- **Different display path**: the i.MX53 QSB ships with its own onboard parallel/LVDS display connector originally paired with a small Freescale-supplied WVGA LVDS panel (the board's stock 4.3" touch panel). If you intend to reuse the MC IMX-LVDS-1 panel from the SABRE-SD plan, verify its interface (LVDS pinout, lane count/voltage, touch controller) matches what the i.MX53's display interface expects — the i.MX53's display controller (DI/LDB block) and connector are not identical to the i.MX6Q's, so don't assume drop-in compatibility. Confirm against both datasheets before wiring it up.

Everything else — Klipper firmware build for the FLY D7, flashing, host-side klippy/Moonraker install, bring-up sequence — is essentially unchanged from the SABRE-SD plan, so this document focuses on what's different: the Yocto machine config and display/performance considerations.

---

## 2. Yocto vs Buildroot for the i.MX53 QSB

Same recommendation as before: **Yocto with meta-freescale**, because `imx53qsb` is a maintained machine definition still present in meta-freescale's current tree (alongside newer i.MX8/9 machines), so you get a known kernel/u-boot baseline rather than bringing up the SoC from scratch. Buildroot remains viable if you want a leaner image (worth considering here specifically *because* the i.MX53 is RAM/CPU constrained — a smaller Buildroot rootfs means less pressure on the single core and 512MB), but you'd own the display bring-up yourself with less prior art than the i.MX6 family has.

If squeezing performance matters more than convenience on this board, Buildroot is a more reasonable trade-off here than it was for the SABRE-SD.

---

## 3. Yocto build steps

### 3.1 Fetch layers
```bash
mkdir ~/imx53-klipper && cd ~/imx53-klipper
git clone -b <release-branch> git://git.yoctoproject.org/poky
git clone -b <release-branch> https://github.com/Freescale/meta-freescale
git clone -b <release-branch> https://github.com/Freescale/meta-freescale-3rdparty
git clone -b <release-branch> https://git.openembedded.org/meta-openembedded
```
As before, pick one current LTS branch name (meta-freescale publishes `scarthgap`, `kirkstone`, etc. per the Yocto release schedule) and use it consistently across all layers.

### 3.2 Initialize build
```bash
source poky/oe-init-build-env build-imx53qsb
bitbake-layers add-layer ../meta-freescale
bitbake-layers add-layer ../meta-freescale-3rdparty
bitbake-layers add-layer ../meta-openembedded/meta-oe
bitbake-layers add-layer ../meta-openembedded/meta-python
bitbake-layers add-layer ../meta-openembedded/meta-networking
```

### 3.3 conf/local.conf
```
MACHINE = "imx53qsb"
ACCEPT_FSL_EULA = "1"
DISTRO ?= "fsl-imx-x11"
IMAGE_INSTALL:append = " python3 python3-pip python3-cffi python3-numpy \
  python3-jinja2 python3-pyserial python3-greenlet libffi \
  git dbus avahi-daemon openssh nano"
```
Given the RAM/CPU budget, consider starting from `core-image-minimal` and adding only what you need (klippy's venv, Moonraker, nginx, KlipperScreen's GTK deps) rather than pulling in `packagegroup-core-x11-sato`'s full desktop — a lighter matchbox/no-window-manager X session is enough to run a single fullscreen KlipperScreen app.

### 3.4 Display device tree
The i.MX53's display block is the DI/IPU-v3 (older generation than i.MX6's), driven through `mxcfb`/`ldb` similarly but with different clocking constraints and a lower max resolution/refresh than the i.MX6Q. If reusing the MC IMX-LVDS-1 panel: get its exact pixel clock, timing, and lane configuration from its datasheet, and check the i.MX53's LDB output capability (single-channel LVDS, lower max pixel clock than i.MX6Q's dual-channel LDB) can actually drive it before wiring — this is the step most likely to fail silently (X11 running, backlight on, no image) if the panel expects more bandwidth than the i.MX53's DI can provide.

### 3.5 Build and flash
```bash
bitbake core-image-minimal   # or core-image-x11 once display bring-up is confirmed
sudo dd if=core-image-minimal-imx53qsb.sdcard of=/dev/sdX bs=1M status=progress conv=fsync
```

---

## 4. FLY D7 firmware + host-side Klipper install

Identical to the SABRE-SD plan (§4–5 there): build Klipper's MCU firmware for STM32F072RBT6 via `make menuconfig`, flash via Mellow's Katapult/USB or CAN-bridge process, then install klippy + Moonraker on the i.MX53 in a Python venv, pointing `[mcu]` at the D7's serial or CAN UUID.

One addition specific to this board: given the constrained CPU, if you plan to use CAN instead of USB for the printer-electronics link, double-check the i.MX53 QSB's available interfaces — it does not have onboard CAN, so you'd need a USB-CAN adapter either way (same as the SABRE-SD, which also lacks onboard CAN).

---

## 5. Bring-up sequence

Same order as the SABRE-SD plan:
1. Boot board standalone, confirm console + network + display before touching Klipper.
2. Bench-test the FLY D7's Klipper firmware in isolation.
3. Wire one axis + thermistor + heater, verify moves and temps.
4. Full wiring per your printer's kinematics.
5. Bring up Moonraker + a lightweight web UI.
6. Bring up KlipperScreen on the panel, watch CPU/RAM headroom closely on this board — if it's tight, drop the web UI (Mainsail/Fluidd) and rely on KlipperScreen + Moonraker's API only, or move the web UI to a separate always-on device.
7. Dry run before printing.

---

## 6. Bottom line

This board *can* work as a Klipper host — the software stack is the same, and `imx53qsb` is still a real Yocto target — but it's a meaningfully weaker machine than the i.MX6Q SABRE-SD. If you're deciding between the two boards for this project rather than committing to both, the SABRE-SD is the safer choice for running klippy + Moonraker + a web UI + KlipperScreen all at once; the i.MX53 QSB is workable if you keep the software footprint lean (KlipperScreen only, no concurrent web UI/webcam streaming) or use it as a display-only front-end talking to a Klipper host running elsewhere.
