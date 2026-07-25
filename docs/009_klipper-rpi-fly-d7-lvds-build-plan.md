# Klipper Host on Raspberry Pi 3B/4 + Mellow FLY D7 + MCIMX-LVDS1 panel — Build Plan

## 1. Why this is a much simpler project than the i.MX6/i.MX53 route

No custom Yocto/Buildroot image, no device-tree bring-up, no kernel cross-compilation. Raspberry Pi OS + KIAUH (the standard community installer) covers Klipper, Moonraker, Mainsail/Fluidd, KlipperScreen, and webcam support (Crowsnest) in one tool, which is the default path almost the entire Klipper community uses. Your remaining real engineering problem is entirely on the display side: bridging the Pi's HDMI or DSI output to the MCIMX-LVDS1 panel's LVDS + capacitive-touch interface.

```
[MCIMX-LVDS1 panel] --LVDS--> [HDMI/DSI-to-LVDS bridge board] --HDMI/DSI--> [Raspberry Pi 3B/4]
        |                                                                        |
    I2C touch --------------------------------------------------------------> GPIO header (I2C)
                                                                                 |
                                                                            USB / CAN
                                                                                 |
                                                                        [Mellow FLY D7 mainboard]
```

---

## 2. Board choice: Pi 3B vs Pi 4

For running klippy + Moonraker + a web UI + KlipperScreen concurrently, **prefer the Pi 4** if you have the option — more RAM and CPU headroom for running KlipperScreen's GTK UI on top of everything else, and dual HDMI if you want the bridge board on one port and a debug monitor on the other during bring-up. A Pi 3B is workable (it's the historically common Klipper host) but keep the software footprint lean: skip GUI extras on the OS itself and let KlipperScreen be the only fullscreen app.

Either way, use **Raspberry Pi OS Lite** (no desktop environment) as the base — KlipperScreen runs its own minimal X session, you don't want a full desktop competing for resources.

---

## 3. Host OS setup

1. Flash Raspberry Pi OS Lite (64-bit) with Raspberry Pi Imager. In the Imager's OS customization options, set hostname, enable SSH, and configure Wi-Fi/locale up front so you don't need a keyboard/monitor for first boot.
2. Boot, SSH in, `sudo apt update && sudo apt full-upgrade -y`.
3. Install KIAUH:
```bash
sudo apt-get install git -y
cd ~ && git clone https://github.com/dw-0/kiauh.git
./kiauh/kiauh.sh
```
4. From KIAUH's menu, install in order: **Klipper → Moonraker → Mainsail (or Fluidd) → KlipperScreen**. KIAUH walks you through Python version, instance count, and printer config paths for each.
5. If you want a webcam later, KIAUH also installs **Crowsnest** — worth doing now while you're already in the tool even if the camera comes later.

---

## 4. FLY D7 firmware + connection

Same as previous plans:
1. `git clone https://github.com/Klipper3d/klipper`, `make menuconfig`: STM32 architecture, STM32F072 processor, bootloader offset matching Mellow's Katapult bootloader, USB or CAN communication.
2. `make` → `out/klipper.bin`.
3. Flash via Mellow's documented Katapult/USB or CAN-bridge process (double-tap reset, confirm `1d50:6177` in `lsusb`, use the flash tool).
4. In `printer.cfg`'s `[mcu]` section, point at `/dev/serial/by-id/...` (USB) or the CAN UUID, as before.
5. Remember: the FLY D7 does not back-power the host — the Pi needs its own 5V supply (official Pi power adapter, or a buck converter off the printer's 24V/12V rail sized for the Pi's actual draw, which is higher than most other SBCs in the space, especially the Pi 4 under load).

---

## 5. Display bridge: HDMI-to-LVDS (recommended over DSI)

Recommended over the DSI route for this project specifically, because:
- The Pi sees the bridge as a plain HDMI monitor — **no special kernel driver, no device-tree overlay, no DSI-specific fiddling.** This matters a lot for a Pi 3B/4 running mainline Raspberry Pi OS, where third-party DSI displays are the more fragile path.
- HDMI-to-LVDS bridge boards are commodity items (built around chips like ADV7613, or combined solutions with an onboard MCU for panel configuration), widely available, and typically support configurable resolution/timing via onboard DIP switches or an EDID-programming step — which you'll want for a 1024×768 XGA panel.

### Steps
1. **Identify your panel's exact interface spec** before buying a bridge: LVDS lane count (likely single-channel for a 1024×768 XGA panel of this generation), bit depth (6-bit or 8-bit per channel), and mapping standard (JEIDA vs VESA). This is on the MCIMX-LVDS1's panel datasheet (the underlying LCD cell, not just NXP's daughtercard page) or can be inferred from the raw LVDS panel it's built on if you can identify that part number by opening the daughtercard.
2. **Buy/build an HDMI-to-LVDS controller board rated for 1024×768** with matching lane count/bit depth. Many generic "universal LVDS driver boards" sold for salvaged laptop/industrial panels support configurable resolution via onboard buttons or resistor-programmed EDID — confirm 1024×768 XGA @ 60Hz is in its supported list, not just assumed.
3. **Wire the panel's LVDS ribbon into the bridge board's LVDS output connector.** This requires matching the MCIMX-LVDS1 daughtercard's panel-side connector pinout to the bridge board's connector — likely not pin-identical, so expect to make a custom ribbon/jumper adapter rather than a direct cable swap. Trace or find documentation for both connectors before wiring power to avoid backfeeding a pin that expects a different voltage.
4. **Power the panel and backlight per its own supply requirements** (usually a separate 12V/5V rail from the bridge board, not pulled through HDMI) — check the panel's backlight driver voltage/current so you don't over/under-drive the LED backlight.
5. **Connect HDMI from the Pi**, boot, and confirm video appears — this should require zero Pi-side configuration if the bridge correctly reports EDID for 1024×768.
6. **If no EDID/auto-negotiation**, force the resolution in `/boot/firmware/config.txt` (Raspberry Pi OS Bookworm+ path; older releases use `/boot/config.txt`):
```
hdmi_group=2
hdmi_mode=35   # 1024x768 60Hz — confirm exact mode number for your firmware version
hdmi_force_hotplug=1
```

### Touch controller
The bridge board handles video only. Wire the panel's capacitive touch controller separately:
1. Identify the touch IC on the MCIMX-LVDS1 daughtercard (open it up or check documentation/silkscreen — commonly a Cypress, Goodix, or similar capacitive touch chip using I2C).
2. Wire its I2C (SDA/SCL) and interrupt line to the Pi's 40-pin header I2C bus (physical pins 3/5 for SDA/SCL, plus a free GPIO for interrupt).
3. Enable I2C in `raspi-config` (`Interface Options → I2C`).
4. Confirm detection with `i2cdetect -y 1` — if the chip shows up at an expected address, Linux likely already has a generic HID/I2C touch driver (`hid-multitouch`, or a vendor-specific `chipname_i2c` driver) that will bind automatically; if not, you'll need to identify the exact part and check kernel driver support before going further.
5. Once recognized as an input device, X11/KlipperScreen should pick it up as a standard touchscreen with no extra configuration — verify calibration/orientation once it's working, since panel rotation vs touch axis mapping commonly needs a manual fix (`xinput --map-to-output` or a udev/xorg.conf calibration matrix).

---

## 6. Bring-up sequence

1. Get HDMI video on the panel with the Pi running stock Raspberry Pi OS, before installing anything Klipper-related — isolate display bring-up from the software stack.
2. Get touch working and calibrated.
3. Install Klipper/Moonraker/Mainsail/KlipperScreen via KIAUH.
4. Bench-test the FLY D7 firmware in isolation (no motors wired), confirm klippy connects.
5. Wire one axis + thermistor + heater, verify `G28`/`G1` and temperature reporting.
6. Full wiring for your kinematics.
7. Confirm KlipperScreen renders correctly on the bridged panel and touch input drives it properly.
8. Dry run before printing.

---

## 7. Where this plan is most likely to need iteration

- **Connector mismatch** between the MCIMX-LVDS1's panel-side ribbon and whatever bridge board you pick — budget time for a custom adapter cable rather than assuming a drop-in fit.
- **EDID/timing mismatch** — if the bridge doesn't correctly report or accept 1024×768@60, you'll be in `config.txt` forcing modes and possibly firmware-updating the bridge board itself.
- **Touch driver identification** — this is the one step that could require real reverse-engineering if the touch chip isn't a common, well-supported part. Worth checking before you commit to this specific panel versus buying a modern all-in-one HDMI touchscreen designed for the Pi ecosystem (which would eliminate this entire section, at the cost of not reusing hardware you already have).
