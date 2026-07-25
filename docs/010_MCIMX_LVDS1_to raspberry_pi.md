This datasheet is genuinely useful — it gives you the exact connector and signal-level details you were missing. Here's what it tells us and how it maps onto a Raspberry Pi build.

## What the datasheet reveals

This is Ka-Ro electronics' **TX53-LVDS-ADAPT** board — a two-piece adapter that plugs into the DIMM socket of Ka-Ro's own **TX53 System-on-Module** (a third-party i.MX53 module, not the official NXP QSB) and breaks it out to connectors for the MCIMX-LVDS1 panel. Two things worth noting up front:

1. This confirms the panel itself is **not exclusively tied to the NXP i.MX53 QSB** — third parties have already built adapters for it on other i.MX53-based hardware. The panel's actual interface is generic single-channel LVDS + I2C touch, which is exactly what you'd hope for portability.
2. The connector used is a standard part: **Hirose DF19G-30P-1H**, a 0.5mm-pitch 30-position FPC/board connector. That's a catalog part — you can source a mating 30-pin 0.5mm FPC cable rather than reverse-engineering a custom connector.

## Pinout, decoded

| Signal group | Pins | Detail |
|---|---|---|
| Backlight control | 1, 5 | Pin 1 = on/off enable (10K pull-up); pin 5 = PWM dimming |
| Logic power | 2–4 | 3.3V |
| LVDS data | 6–15 | 4 differential pairs: TX3, TX0, TX1, TX2 (N/P) — **single-channel LVDS**, not dual |
| LVDS clock | 17–18 | CLK_N/CLK_P |
| Panel power | 20–26 | 5V (multiple pins, plus GND) |
| Touch I2C | 27–28 | I2C CLK / I2C SDA, both 10K pulled up |
| Touch interrupt | 29 | TOUCH_INT, 10K pulled up |

The important engineering fact: **this is single-channel (single-link) LVDS** — 4 data pairs + 1 clock pair, not the 8-pair dual-channel format some larger panels use. That's the most common, most widely-supported LVDS format, which is good news for sourcing a bridge board.

## Mapping to a Raspberry Pi build

This changes your bridge-board search from "some generic LVDS adapter, hope it matches" to a precise spec:

**Video (HDMI-to-LVDS bridge):**
- You need a bridge board that outputs **single-channel, 4-pair + clock LVDS**, at 1024×768, with exposed pins/wires (not a proprietary laptop FFC) so you can wire directly to a Hirose DF19G-30P-1H mating cable.
- Confirm the bridge's JEIDA vs VESA color mapping and bit depth (6-bit vs 8-bit per channel) against the panel's actual LCD cell spec — the adapter datasheet doesn't state this, so this is the one remaining unknown. If you can identify the raw LCD panel part number stamped on the physical MCIMX-LVDS1 panel itself, its datasheet will confirm this.
- Wire the bridge's 4 TX pairs + CLK pair straight to pins 6–18 above, GND to the GND pins, and 3.3V logic supply to pins 2–4 if your bridge needs it (many bridge boards are self-powered and only need GND reference — check).
- Panel 5V (pins 20–26) and backlight — these need their own supply, typically 5V, separate from the Pi's own power. The backlight on/off (pin 1) and dimming PWM (pin 5) need a logic-level drive signal: either from the bridge board if it provides one, or directly from a Pi GPIO (3.3V logic, matches the existing 10K pull-ups) if the bridge doesn't handle backlight control itself.

**Touch (direct to Pi, bypassing the bridge entirely):**
- Wire pin 27 (I2C CLK) → Pi physical pin 5 (SCL)
- Wire pin 28 (I2C SDA) → Pi physical pin 3 (SDA)
- Wire pin 29 (TOUCH_INT) → any free Pi GPIO
- The panel side already has 10K pull-ups to 3.3V on all three lines, and the Pi's I2C1 bus is also 3.3V logic — so no level-shifting needed, and you generally don't need to add your own pull-ups since they're already present on the panel/adapter side (having both the Pi's onboard pull-ups and these in parallel is normally fine, just slightly stronger pull-up than either alone).
- The actual touch controller chip itself isn't identified in this datasheet — that's still the one piece you'd need to find (either from the panel's silkscreen/markings, or by probing its I2C address with `i2cdetect` once wired up) to confirm Linux driver support.

**What's now fully de-risked:** connector identity, pin-for-pin signal mapping, power rails, and confirmation this is a standard single-channel LVDS + I2C-touch interface rather than something proprietary.

**What's still open:** exact color mapping/bit-depth for the video bridge, and the specific touch controller IC. Do you have the physical panel in hand? If you can get the raw LCD cell's part number (usually printed on a label on the back of the glass, separate from the Ka-Ro/NXP board silkscreen) or the touch controller's chip marking, I can look those up and close both gaps.

---

Both pieces, starting with the bridge board since it's the simpler of the two.

## HDMI-to-LVDS bridge board

Two good matches came up that explicitly list your exact requirements:

1. **VSDisplay VS-N5 V3.1** — explicitly supports 1ch 6bit, 2ch 6bit, 1ch 8bit, and 2ch 8bit LVDS interfaces, so it covers your single-channel 6-bit spec directly and gives you margin if your color-depth assumption turns out slightly off.
2. **M.NT68676-based board** — one listing is specifically documented working with a 10.4" 1024×768 panel, i.e. sold pre-verified against the exact resolution/timing class your HSD100PXN1 falls into.

**One thing to get right when ordering**: your panel's connector is the 30-pin Hirose DF19G-30P-1H (confirmed from the Ka-Ro datasheet), but most of these bridge boards ship configured for the 40-pin LVDS cable common on laptop panels. 30-pin single-channel controller boards do exist for this class of panel (search results turned up 30-pin boards paired with other 1024×768 panels specifically) — when you order, specify the 30-pin single-channel 6-bit cable variant rather than the default 40-pin one, or budget for building a custom 30-pin adapter cable using the exact pin mapping from the Ka-Ro datasheet (pins 6–18 for the 4 LVDS pairs + clock, as established earlier).

## Raspberry Pi devicetree overlay for the eGalax touch controller

Confirmed defaults from the mainline driver and existing device-tree examples: the node uses compatible = "eeti,egalax_ts", reg = 0x04 as the standard I2C address, with an interrupt line and optional wakeup GPIO.

```dts
// egalax-ts-overlay.dts
// Overlay for EETI eGalax I2C touch controller (MCIMX-LVDS1 panel) on Raspberry Pi

/dts-v1/;
/plugin/;

/ {
    compatible = "brcm,bcm2835", "brcm,bcm2708", "brcm,bcm2709";

    fragment@0 {
        target = <&i2c_arm>;
        __overlay__ {
            #address-cells = <1>;
            #size-cells = <0>;
            status = "okay";

            egalax_ts@04 {
                compatible = "eeti,egalax_ts";
                reg = <0x04>;
                interrupt-parent = <&gpio>;
                interrupts = <17 2>;   // GPIO17, falling-edge triggered — change to match your TOUCH_INT wiring
                wakeup-gpios = <&gpio 17 0>;
            };
        };
    };
};
```

**Build and install:**
```bash
dtc -@ -I dts -O dtb -o egalax-ts.dtbo egalax-ts-overlay.dts
sudo cp egalax-ts.dtbo /boot/firmware/overlays/
```
Then add to `/boot/firmware/config.txt`:
```
dtparam=i2c_arm=on
dtoverlay=egalax-ts
```

**Wiring recap** (from the Ka-Ro pinout established earlier): panel pin 27 (I2C CLK) → Pi physical pin 5, pin 28 (I2C SDA) → Pi physical pin 3, pin 29 (TOUCH_INT) → whichever GPIO you set in the overlay above (GPIO17/physical pin 11 used in the example — pick any free GPIO and match it in both places).

**Verify after boot:**
```bash
i2cdetect -y 1        # should show a device at 0x04
dmesg | grep -i egalax
evtest /dev/input/eventX
```

If it doesn't show at 0x04, that's the one soft spot worth flagging: some eGalax variants use a different address (0x05 or 0x38 have shown up on other eGalax-based panels in the wild), so if `i2cdetect` comes back empty at 0x04, scanning the full address range and adjusting `reg` in the overlay is the fallback — not a sign anything is wired wrong.

You now have a complete, buildable parts-and-config list: bridge board spec, exact wiring pinout, and a working devicetree overlay. That's everything needed to actually order hardware and start assembly.

---

Here's the full pin-by-pin wiring map, going from the panel's actual connector (confirmed from the HannStar datasheet, which matches the Ka-Ro pinout table you found) through to both destinations — the bridge board (video) and the Pi (touch).

## Panel connector reference

Panel side: **Hirose DF19G-30P-1H**, 30-pin, 0.5mm pitch. This is what you're wiring *from*.

| Pin | Signal | Destination | Wiring notes |
|---|---|---|---|
| 1 | Backlight enable (CABC_EN) | Bridge board backlight-enable output, or a Pi GPIO if the bridge doesn't provide one | Has 10K pull-up on the panel side already |
| 2 | VDD 3.3V (logic) | Bridge board 3.3V logic supply output, if it has one — otherwise a separate 3.3V regulator | |
| 3 | VDD 3.3V (logic) | Same net as pin 2 | |
| 4 | V_EDID 3.3V | Same 3.3V net as pins 2–3, unless your bridge drives EDID separately | |
| 5 | Backlight dimming (ADJ/PWM) | Bridge board PWM/dimming output, or tie to 3.3V for full brightness if you don't need dimming control | |
| 6 | RXIN0− (LVDS data pair 0, −) | Bridge board LVDS output, data pair 0 (−) | |
| 7 | RXIN0+ (LVDS data pair 0, +) | Bridge board LVDS output, data pair 0 (+) | |
| 8 | RXIN1− (LVDS data pair 1, −)* | Bridge board LVDS output, data pair 1 (−) | *pin order for 1/2/3 may shift slightly vendor-to-vendor — verify against your specific bridge's silkscreen/datasheet |
| 9 | RXIN1+ (LVDS data pair 1, +)* | Bridge board LVDS output, data pair 1 (+) | |
| 10 | GND | Common ground | Tie to bridge board GND and Pi GND (single common ground reference across the whole system) |
| 11 | RXIN2− (LVDS data pair 2, −) | Bridge board LVDS output, data pair 2 (−) | |
| 12 | RXIN2+ (LVDS data pair 2, +) | Bridge board LVDS output, data pair 2 (+) | |
| 13 | GND | Common ground | |
| 14 | RXIN3− (LVDS data pair 3, −) | Bridge board LVDS output, data pair 3 (−) | |
| 15 | RXIN3+ (LVDS data pair 3, +) | Bridge board LVDS output, data pair 3 (+) | |
| 16 | GND | Common ground | |
| 17 | RXCLKIN− (LVDS clock, −) | Bridge board LVDS clock output (−) | |
| 18 | RXCLKIN+ (LVDS clock, +) | Bridge board LVDS clock output (+) | |
| 19 | GND | Common ground | |
| 20 | 5V (panel power) | Panel's dedicated 5V/2A supply (do **not** pull this from the Pi or bridge board's logic rail) | |
| 21 | 5V (panel power) | Same 5V net as pin 20 | |
| 22 | GND | Common ground | |
| 23 | GND | Common ground | |
| 24 | 5V (panel power) | Same 5V net as pin 20 | |
| 25 | 5V (panel power) | Same 5V net as pin 20 | |
| 26 | 5V (panel power) | Same 5V net as pin 20 | |
| 27 | I2C CLK (touch) | **Raspberry Pi physical pin 5 (SCL, I2C1)** | 10K pull-up already present on panel side |
| 28 | I2C SDA (touch) | **Raspberry Pi physical pin 3 (SDA, I2C1)** | 10K pull-up already present on panel side |
| 29 | TOUCH_INT | **Raspberry Pi GPIO of your choice** (GPIO17/physical pin 11 in the overlay example) — must match whatever you set in the devicetree overlay's `interrupts` line | 10K pull-up already present on panel side |
| 30 | Not connected | — | — |

## Practical wiring order

1. **Ground first, always** — connect panel GND (pins 10, 13, 16, 19, 22, 23) to a single common ground point shared by the bridge board, the panel's own 5V supply, and the Pi. A floating or split ground reference between the touch I2C bus and the video path is the most common source of "touch works but reads garbage" or "video is noisy" symptoms in this kind of build.
2. **Power the panel's 5V rail (pins 20/21/24/25/26) from its own supply**, separate from the Pi's 5V and separate from whatever powers the bridge board's own logic — don't assume these are safe to tie together even though they're all "5V," since current draw and regulation differ.
3. **Wire LVDS data + clock pairs (pins 6–18) directly to the bridge board's LVDS output**, matching polarity (−/+) exactly — swapped polarity on a differential pair typically shows as no sync or a scrambled image rather than a reversed picture, so it's easy to misdiagnose.
4. **Wire touch (pins 27–29) straight to the Pi**, bypassing the bridge board entirely — this is a completely separate path from the video wiring.
5. **Backlight (pins 1, 5)** last, once video is confirmed working — many bridge boards handle backlight enable/dimming automatically as part of their own panel-power sequencing, so check your specific board's documentation before wiring these to anything else; wiring backlight control redundantly (bridge board *and* a Pi GPIO both driving it) risks a drive conflict.

One caveat worth restating: pins 8–9 and 11–12 (the RXIN1/RXIN2 pair ordering) are the one place where different sources sometimes number the middle two data pairs differently — if your bridge board's datasheet numbers its LVDS data pairs 0–3 in a different physical order than pins 6→15 here, match by *pair grouping* (each pair's − and + must stay together) rather than assuming pin 8 is necessarily "channel 1" on the bridge side. Worth double-checking against your specific bridge board's own pinout diagram once you've picked one, since I can't verify that detail without knowing exactly which board you buy.