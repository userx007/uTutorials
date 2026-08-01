# G-Code & CNC Machine Fundamentals: A Complete Tutorial

## Table of Contents
1. What Is G-Code?
2. Coordinate Systems: Machine Space vs. Work Space
3. Axes and the Right-Hand Rule
4. Homing and Reference Points
5. G-Code Program Structure
6. Positioning Modes: Absolute vs. Incremental
7. Units and Precision
8. Common G-Codes Reference
9. Common M-Codes Reference
10. Work Coordinate Offsets (G54–G59)
11. Feed Rate, Spindle Speed, and Tool Selection
12. Tool Length and Radius Compensation
13. Canned Cycles
14. Soft Limits, Hard Limits, and Safety
15. Reading a Full Example Program
16. Common Pitfalls and Best Practices

---

## 1. What Is G-Code?

G-code (RS-274) is the numerical control (NC) programming language that tells a CNC (Computer Numerical Control) machine **where to move, how fast to move, and what to do** at each point along a toolpath. It originated in the 1950s at MIT for early NC milling machines and was standardized by EIA/ISO. Despite decades of new machine technology, G-code (or close dialects of it) remains the near-universal language for 3-axis mills, lathes, routers, plasma cutters, laser cutters, and 3D printers.

At its core, G-code is a line-by-line list of instructions. Each line (called a **block**) contains one or more **words** — a letter followed by a number — that together define a single action: move to a position, turn on the spindle, change tools, pause, etc.

```
G1 X10.0 Y5.0 F300
```
This line says: "Move in a straight line (G1) to X=10.0, Y=5.0, at a feed rate of 300 units/min."

There are two broad families of codes:
- **G-codes (preparatory codes):** define motion type, coordinate system, units, plane selection, etc.
- **M-codes (miscellaneous/machine codes):** control non-motion machine functions — spindle on/off, coolant, program stop, tool change.

Different machine controllers (Fanuc, Haas, Siemens, LinuxCNC, GRBL, Marlin) implement slightly different dialects, but the core codes below are nearly universal.

---

## 2. Coordinate Systems: Machine Space vs. Work Space

This is one of the most important — and most confusing — concepts for beginners. A CNC machine actually tracks position in **two overlapping coordinate systems**.

### Machine Coordinate System (MCS) — also called "Machine Space"
- A fixed coordinate system built into the machine itself, with its origin (**Machine Zero**, or **Machine Home**, symbol `G53` context) typically at a mechanical hard-stop position — often the extreme corner of travel (e.g., far back-right-top of a mill's work envelope).
- This origin **never moves**. It is established once, physically, by the machine builder, and rediscovered every session via **homing**.
- All other coordinate systems are defined as offsets *from* this fixed machine origin.
- You rarely program directly in machine coordinates, but you can explicitly command a move in machine space using `G53` (a non-modal code) — e.g., `G53 G0 Z0` to send the Z axis to its home/safe position regardless of any active work offset.

### Work Coordinate System (WCS) — also called "Workspace" or "Part Coordinate System"
- A user-defined coordinate system whose origin ("**Work Zero**" or "**Part Zero**") is set wherever is convenient for the job — typically a corner or center of the raw stock/workpiece.
- Established by touching off the tool against the material and recording an **offset** (the distance from Machine Zero to Work Zero) into an offset register, commonly `G54` through `G59` (and extended sets like `G54.1 P1`...`P48` on Fanuc-style controls).
- All ordinary program coordinates (`X`, `Y`, `Z` values in your G-code) are interpreted relative to whichever WCS is currently active.
- This lets you write a program once and reuse it on different machines, or run the same program for multiple identical parts (fixture offsets) just by changing which work offset is active or where it points.

### Why two systems?
Think of Machine Space as the immovable "master ruler" bolted to the machine, and Work Space as a "sticky note ruler" you can reposition anywhere on the table for each job. The controller always knows both simultaneously:

```
Work Position = Machine Position − Work Offset
```

If you jog the machine or run `G53` moves, you're working in Machine Space. As soon as a work offset (`G54`, etc.) is active and you issue normal `G0`/`G1` moves, you're working in Work Space.

### Local/Temporary Offsets
On top of G54–G59, most controls also support:
- **G92 (or G10 L20 on Fanuc)** — a programmable, temporary origin shift, often used to "zero" a position mid-program.
- **Tool offsets** — a per-tool Z (and sometimes X/Y) adjustment so different tool lengths don't require re-touching-off the part zero.

---

## 3. Axes and the Right-Hand Rule

Standard milling machines use three linear axes — **X, Y, Z** — plus optional rotary axes **A, B, C** (rotation around X, Y, Z respectively).

- **Z axis**: always aligned with the spindle (the tool axis). Positive Z moves the tool *away* from the workpiece (up, on a vertical mill).
- **X axis**: typically the longer horizontal travel — left/right when facing the machine.
- **Y axis**: the other horizontal travel — front/back.

Orientation follows the **right-hand rule**: point your right thumb along +Z, and your curled fingers show the positive rotational direction (used for arcs and rotary axes). Equivalently, index finger = +X, middle finger = +Y, thumb = +Z, using your right hand.

For **lathes**, the convention differs: Z runs along the spindle axis of rotation, and X is the radial/diameter direction (X is often programmed as diameter, not radius — controller-dependent).

---

## 4. Homing and Reference Points

**Homing** (also called "referencing" or "zero-return") is the process by which the machine rediscovers where Machine Zero physically is, since most CNC axes use incremental encoders or open-loop steppers that lose absolute position when powered off.

### How homing works
1. On startup, the controller does **not** know its position.
2. The operator initiates a homing cycle (often a dedicated button, or `$H` in GRBL).
3. Each axis drives toward a **limit/home switch** at a defined edge of travel.
4. Once the switch triggers, the axis backs off slightly and re-approaches slowly for repeatable accuracy (a "seek and slow re-touch" routine).
5. The controller sets that precise position as **Machine Zero** (X0 Y0 Z0 in machine coordinates) or as a known fixed offset from it.
6. All active work offsets (G54, etc.) are then recalculated relative to this newly established Machine Zero.

### Types of reference points
- **Machine Zero (M):** the fixed origin of the whole machine, set via homing.
- **Work Zero / Part Zero (W):** the origin of the current WCS, set by the operator via a touch-off procedure (touching the tool to a known point on the stock and zeroing that axis, or using a touch probe).
- **Tool Change Position / Reference Position:** a safe, fixed position (often near or at Machine Zero) the machine retracts to before changing tools — reached via `G28` (return to reference point) or `G30` (secondary reference point) on Fanuc-style controls, or `G53` moves on others.

### Why homing matters
- Establishes the safe travel envelope (**soft limits** — see Section 14 — are usually calculated relative to Machine Zero, so they only work correctly after homing).
- Restores repeatability: if the machine loses power mid-job, homing lets it re-align to the exact same physical reference every time.
- A machine that has not been homed typically cannot safely run G-code, because the controller has no reliable idea where the tool actually is relative to the table or fixtures.

---

## 5. G-Code Program Structure

A typical program is a plain-text file (extension `.nc`, `.gcode`, `.tap`, `.cnc`) structured like this:

```
%                      (optional program start delimiter - older Fanuc controls)
O1001                  (program number, Fanuc-style)
G21 G17 G90 G94         (setup block: mm, XY plane, absolute mode, feed/min)
G54                     (select work coordinate system 1)
G0 Z25.0                (rapid to safe height)
M3 S8000                (spindle on clockwise, 8000 RPM)
G0 X0 Y0                (rapid to start position)
G1 Z-2.0 F150            (plunge into material)
G1 X50.0 F400            (linear cut)
G2 X70.0 Y20.0 I20.0 J0  (clockwise arc)
G0 Z25.0                (retract)
M5                      (spindle off)
M30                     (end of program, rewind)
%                      (program end delimiter)
```

Key structural conventions:
- **Line numbers (`N10`, `N20`...)** are optional labels, mainly used for editing/searching or `M99 P` jumps — not required by most modern controllers.
- **Comments** are enclosed in parentheses `(like this)` or after a semicolon `; like this` depending on dialect.
- **Modal vs. non-modal codes:** *Modal* codes (like `G1`, `G90`, `G54`) stay active until changed by another code in the same group — you don't need to repeat them every line. *Non-modal* codes (like `G4` dwell, `G28`, `G53`) apply only to the block they're on.
- Blocks are typically executed top to bottom, sequentially, unless a subroutine call (`M98`/`M99`) or canned cycle alters flow.

---

## 6. Positioning Modes: Absolute vs. Incremental

- **G90 — Absolute positioning:** every coordinate is measured from the active work origin. `X10 Y10` always means the same physical point regardless of where the tool currently is.
- **G91 — Incremental positioning:** every coordinate is a *delta* from the tool's current position. `X10 Y10` means "move 10 units further in X and 10 further in Y from wherever you are now."

Most professional programs run almost entirely in **G90 (absolute)** because it's less error-prone — a single bad line in incremental mode compounds into every subsequent move.

---

## 7. Units and Precision

- **G20** — Inch mode.
- **G21** — Millimeter mode.

This should always be set explicitly at the top of a program — mixing it up (sending mm coordinates while the machine is in inch mode) is a classic and dangerous cause of crashes, since a "10" becomes either 10 mm or 254 mm depending on context.

Precision/decimal handling and trailing-zero rules vary slightly by controller, but nearly all modern controls accept plain decimal notation (`X12.345`).

---

## 8. Common G-Codes Reference

| Code | Meaning |
|------|---------|
| `G0` | Rapid positioning (fastest possible move, non-cutting, straight-line-ish but not guaranteed linear on all controls) |
| `G1` | Linear interpolation (straight-line cutting move at programmed feed rate) |
| `G2` | Circular interpolation, clockwise |
| `G3` | Circular interpolation, counter-clockwise |
| `G4` | Dwell (pause) — e.g. `G4 P2.0` pauses 2 seconds |
| `G17/G18/G19` | Select working plane: XY / XZ / YZ (affects arc and cutter-comp behavior) |
| `G20/G21` | Inch / Millimeter units |
| `G28` | Return to reference/home position (often via an intermediate point) |
| `G40/G41/G42` | Cutter compensation off / left / right |
| `G43/G49` | Tool length compensation on (with offset number) / off |
| `G54–G59` | Select work coordinate system 1–6 |
| `G80` | Cancel canned cycle |
| `G81–G89` | Canned cycles (drilling, tapping, boring, etc.) |
| `G90/G91` | Absolute / incremental positioning |
| `G92` | Set/shift work coordinate offset at current position |
| `G94/G95` | Feed per minute / feed per revolution |
| `G53` | Move in machine coordinates (non-modal, one-shot) |

*(Exact code sets vary — always check your specific controller's manual, especially for hobbyist firmwares like GRBL or Marlin, which support a subset plus their own extensions.)*

---

## 9. Common M-Codes Reference

| Code | Meaning |
|------|---------|
| `M0` | Program stop (unconditional pause, waits for operator) |
| `M1` | Optional stop (pauses only if operator has enabled "optional stop") |
| `M2`/`M30` | End of program (M30 also rewinds to the start) |
| `M3` | Spindle on, clockwise |
| `M4` | Spindle on, counter-clockwise |
| `M5` | Spindle stop |
| `M6` | Tool change (usually paired with a `T` word, e.g. `T2 M6`) |
| `M7`/`M8` | Coolant on (mist / flood) |
| `M9` | Coolant off |
| `M98`/`M99` | Call subroutine / return from subroutine |

---

## 10. Work Coordinate Offsets (G54–G59)

Modern controls store multiple work offsets simultaneously so an operator can:
- Machine several identical parts held in different fixtures without rewriting the program — just switch `G54` → `G55` → `G56`, etc.
- Keep a saved zero point for a job that can be recalled instantly next time the same fixture is loaded.

**Setting a work offset** (typical manual workflow):
1. Home the machine (establishes Machine Zero).
2. Jog the tool to touch a known reference point on the stock (e.g., front-left corner, top surface).
3. Tell the controller "this position is X0 Y0 Z0 for G54" — the controller then calculates and stores the Machine-to-Work offset automatically.
4. From then on, any program using `G54` and coordinates like `X10 Y10` will correctly reference that physical corner.

Extended offset systems (Fanuc `G54.1 P1`...`P300`, etc.) exist on machines needing many simultaneous fixtures.

---

## 11. Feed Rate, Spindle Speed, and Tool Selection

- **F (Feed rate):** how fast the tool moves while cutting, in units/min (`G94`, the default) or units/rev (`G95`, common on lathes). `G0` rapid moves ignore F and move at the machine's maximum rapid rate.
- **S (Spindle speed):** RPM, e.g. `S8000`. Direction is set separately via `M3`/`M4`.
- **T (Tool number):** selects a tool from the magazine/carousel, e.g. `T3`. Actually swapping the tool into the spindle usually requires `M6` as well (`T3 M6`).

---

## 12. Tool Length and Radius Compensation

- **Tool Length Offset (G43 Hxx / G49):** Since every tool has a different length, the control adds a stored per-tool Z offset so the same Z-zero in the program works no matter which tool is loaded. `G43 H1` applies offset register 1; `G49` cancels it.
- **Cutter Radius Compensation (G41/G42/G40):** Lets you program a toolpath along the actual desired part edge, while the control automatically offsets the path left (`G41`) or right (`G42`) by the tool's radius, so you don't have to manually calculate offset paths for every tool diameter. `G40` cancels compensation.

---

## 13. Canned Cycles

Canned cycles are pre-packaged, repeatable motion sequences (mostly for drilling/tapping/boring) that collapse many lines of G-code into one, and repeat automatically at multiple X/Y locations.

| Code | Cycle |
|------|-------|
| `G81` | Simple drilling |
| `G82` | Drilling with dwell (for spot-facing) |
| `G83` | Peck drilling (deep holes, chip-breaking retracts) |
| `G84` | Tapping cycle |
| `G85`/`G86` | Boring cycles |
| `G80` | Cancel any active canned cycle |

Example — three peck-drilled holes:
```
G99 G83 X10 Y10 Z-20 R2 Q5 F150
X30 Y10
X50 Y10
G80
```
Each subsequent `X`/`Y` line reuses the same cycle parameters (Z depth, retract plane, peck increment) at a new location — a major efficiency win over writing full move sequences for each hole.

---

## 14. Soft Limits, Hard Limits, and Safety

- **Hard limits:** physical limit switches that trigger an emergency stop if the machine tries to travel past the mechanical edge of an axis. Protect against gross crashes but stop motion abruptly.
- **Soft limits:** software-defined travel boundaries, calculated from Machine Zero once the machine is homed. The controller refuses to execute any move that would exceed them, checked *before* motion starts — a gentler, more predictable safeguard than a hard limit trip.
- Because soft limits depend on knowing the true Machine Zero, **they are typically only enforced after a successful homing cycle** — another reason homing is mandatory at the start of every session.
- Good practice: always run a program in "air cut" / dry-run mode or with a graphical simulation first, keep a hand on the feed-hold/E-stop, and verify the work offset is correct before the first real cut.

---

## 15. Reading a Full Example Program

```
G21 G17 G90 G94        ; mm, XY plane, absolute, feed/min
G54                     ; select work offset 1
G0 Z25.0                ; rapid to safe height
M3 S10000                ; spindle on CW at 10,000 RPM
G0 X0 Y0                ; rapid to part origin in XY
G1 Z-3.0 F200            ; plunge to depth
G1 X40.0 F500             ; cut along X
G1 Y30.0                  ; cut along Y
G2 X40.0 Y30.0 I-20.0 J0  ; example arc (illustrative)
G1 X0 Y0                  ; cut back to origin
G0 Z25.0                 ; retract
M5                       ; spindle off
G28 G91 Z0                ; return to home in Z (incremental, one-shot)
G90                      ; restore absolute mode
M30                      ; end program
```

Walking through it: units and modes are set once at the top; the tool rapids to a safe clearance height; the spindle starts; the tool rapids over the start point in X/Y only (keeping Z safe); then plunges and cuts the profile at a controlled feed rate; retracts; stops the spindle; returns home; and ends the program.

---

## 16. Common Pitfalls and Best Practices

- **Always confirm units (G20/G21) match your CAM post-processor output** — the single most common cause of catastrophic crashes.
- **Never skip homing** at the start of a session, even if the machine "seems" positioned correctly.
- **Double-check which work offset is active** before running a job, especially on multi-fixture setups.
- **Keep Z retracts generous** when moving between operations to clear clamps and stock.
- **Use dry-run/simulation** in your CAM software or controller before cutting real material.
- **Understand modal state** — a `G1` on one line stays active on the next line even without repeating it, which is efficient but means a missing feed rate change may not do what you expect.
- **Comment your code** liberally, especially tool changes, offsets, and depth values, for future readability.

---

This covers the essential mental model: G-code is a sequence of modal instructions; every machine tracks a fixed **Machine Space** anchored by **homing**, and a movable, user-defined **Work Space** anchored by touch-off offsets (G54–G59); and the vocabulary of G/M codes builds up from there into real programs. From here, the best way to build fluency is to read post-processed output from CAM software (Fusion 360, Mastercam, etc.) side-by-side with the toolpath preview, and to practice manual "conversational" programming on simple 2D shapes.

Here's the machine space vs. work space relationship in ASCII:

```
 Machine space (fixed — set by homing)
 +-----------------------------------------------------+
 |                                                     |
 |  (Machine Zero / Home)                              |
 |   X-------.                                         |
 |            \                                        |
 |             \   offset (G54)                        |
 |              \                                      |
 |               \        Work space                   |
 |                \       +----------------------+     |
 |                 `----->X                      |     |
 |                        | (Work Zero)          |     |
 |                        |                      |     |
 |                        |      STOCK / PART    |     |
 |                        |                      |     |
 |                        +----------------------+     |
 |                                                     |
 |  Y                                                  |
 |  ^                                                  |
 |  |                                                  |
 |  +----> X                                           |
 +-----------------------------------------------------+
```

And the axis directions with the right-hand rule:

```
   Top view (table plane)              Side view (spindle axis)

        Y+                                   Z+
        ^                                     ^
        |                                     |   (away from
        |                                     |    workpiece,
        |                                     |    retract)
        |                                     |
   (0,0)o------> X+                     ------o------  <- table / part
        |                                     |
                                              |
                                              v
                                              Z-
                                        (plunge into material)

   Right-hand rule:
     thumb  -> points along +Z
     fingers -> curl in the direction of
                positive (CCW) rotation
```

These are the same relationships as the rendered diagrams — machine zero fixed by homing, work zero offset from it, and Z+ always pointing away from the part while X/Y define the table plane.