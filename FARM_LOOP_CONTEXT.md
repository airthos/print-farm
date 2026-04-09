# Bambu P1S Farm Loop — Project Context

This document gives Claude Code full context on the farm loop automation project: hardware setup, architecture decisions, hard-won lessons, and the current state of the codebase. Read this before making any changes.

---

## Hardware

- **Printer**: Bambu Lab P1S (one or more units)
- **Ejection hardware**: FarmLoop Stage 1 clip — mounted at the front-left corner of the bed. Flexes the spring steel PEI plate to release prints. Clip engages at **Z204** (26 × 2mm jog clicks up from the Z256 bed floor).
- **Bed**: Textured PEI plate. Releases well at 25°C.
- **Doors**: Must be removed for the push sweep to reach the front of the bed.
- **Host machine**: Ubuntu 24.04 desktop running BambuBuddy in Docker.
- **Remote access**: Tailscale for SSH from Mac.

---

## Goal

A reliable automated print loop:

1. BambuStudio slices the job and exports a `.gcode.3mf`
2. `farm_loop.py` post-processes the file (strips stock end gcode, injects the farm loop end sequence)
3. The modified `.gcode.3mf` is sent to the printer via BambuBuddy
4. After printing, the end sequence runs: cooldown → bed flex → push sweeps → part falls off
5. The next job starts automatically via BambuBuddy queue

---

## The Script: `farm_loop.py`

### What it does

Takes a BambuStudio-exported `.gcode.3mf` as input. Two modes:

**Normal mode** (default): Strips the stock end gcode, injects the farm loop end sequence, repacks and outputs a new `_farmed.gcode.3mf`.

**Test mode** (`--test`): Strips everything after `M204 S10000` in the machine start block, injects a `G28` home and 10s dwell, then the full end sequence. Use to validate push/flex motion with a physical object on the bed.

### Usage

```bash
# Normal mode
python farm_loop.py your_print.gcode.3mf

# With options
python farm_loop.py your_print.gcode.3mf --cooldown-temp 25 --push-speed 300

# Test mode
python farm_loop.py your_print.gcode.3mf --test

# Custom output
python farm_loop.py your_print.gcode.3mf -o output.gcode.3mf
```

### Options

| Flag | Default | Description |
|---|---|---|
| `--cooldown-temp` | 25 | Bed temp (°C) to wait for before ejection |
| `--push-speed` | 300 | Center lane push feedrate mm/min (slow, Factorian default) |
| `--push-x` | auto | X centre of push sweeps. Auto-detected from file if omitted |
| `--lane-offset` | 60 | mm left/right of push_x for side lanes |
| `--flex-cycles` | 3 | FarmLoop Stage 1 flex cycles before push (0 to disable) |
| `--flex-z` | 204 | Absolute Z where FarmLoop clip engages (26 × 2mm jog clicks from Z256 floor) |
| `--flex-drop` | 20 | mm bed drops past clip per flex cycle (10 × 2mm jog clicks) |
| `-o` / `--output` | auto | Output file path |
| `--test` | off | Test mode flag |

### Output filenames (auto)

- Normal: `{stem}_farmed.gcode.3mf`
- Test: `{stem}_test.gcode.3mf`

---

## End Sequence (FactorianDesigns V2)

Based directly on `Print_Automation_Costum_Codes_by_FactorianDesigns_Version2/P1 Codes/End_Code_Automatic_Pushoff_P1_V2.txt`.

Order of operations:

1. **Retract** — `G92 E0`, `G1 E-0.8 F1800`, raise Z slightly (`max_z + 0.5`)
2. **Safe position** — `G1 X65 Y245 → Y265` (rear wipe zone, x2 for good measure)
3. **Fans off** — M140 S0, M106 S0/P2/P3
4. **AMS filament retract** — `M620 S255` → move to X20 Y50 → `G1 Y-3` → `T255` → `M621 S255` → `M104 S0`
5. **Timelapse end** — `M622.1 S1` → `M991 S0 P-1` block
6. **M17 S** — save motor currents (Factorian legacy, harmless)
7. **Cooldown Z** — `M17 Z0.4`, `G1 Z{push_z} F600` (dynamic — same as push height, safe for any print height)
8. **Active cooling fans** — `M106 P2 S255`, `M106 P3 S200`
9. **Wait for bed temp** — `M190 S{cooldown_temp}` × 40 lines (Factorian's timeout workaround — 40 × 90s = 60min max, completes instantly once temp is reached)
10. **Fans off**
11. **FarmLoop Stage 1 bed flex** — toolhead stays at X65 Y265, bed cycles Z204 → Z224 → Z204 × 3 (clip engages at Z204, flexes to Z224 to break print free)
12. **Push Z** — `G1 Z{push_z}`: `max_z > 31 → max_z - 30`, else `Z1` (Factorian formula)
13. **Center push** — `G1 X{push_x} Y230 F1200` → `Y25 F{push_speed}` (slow)
14. **Right lane** — `X{push_x+60} Y200 → Y25 F2000`
15. **Left lane** — `X{push_x-60} Y200 → Y25 F2000`
16. **Reset** — `M220 S100`, `M201.2 K1.0`, `M73.2 R1.0`, `M1002 set_gcode_claim_speed_level : 0`
17. **Motor current** — `M17 X0.8 Y0.8 Z0.5`, `M400`, `M73 P100 R0`

### Push X auto-detection priority

1. `--push-x` manual override
2. `print_x_min` / `print_x_max` from GCODE header comments
3. `bbox_objects[0].bbox` from `Metadata/plate_1.json` inside the 3MF (most accurate)
4. X=120 fallback (Factorian's default)

---

## Bed Placement Rules (from Factorian)

- **Front strip Y0–25**: No parts, no purge line. Push sweeps end at Y25.
- **Rear strip Y245–265**: Wipe/park zone. Keep clear.
- **Front-left corner**: FarmLoop Stage 1 clip. Set exclusion zone in BambuStudio at slice time.
- **Purge line**: Must be disabled. Factorian comments it out entirely in the start code (nozzle load line section). Disable in BambuStudio: Print Settings → Others → disable front purge.
- **Multi-object push order**: Always right to left (higher X first) to avoid LIDAR body hitting already-pushed parts.

---

## Critical Architecture Decisions

### Must input a `.gcode.3mf`, not a plain `.gcode`

The printer (P1S) only accepts `.gcode.3mf` files over the network. Plain `.gcode` files produce error `0500-4003`.

### Must repack the original zip, not generate a new one

The Bambu firmware parser is strict about the zip's internal structure — file order, per-file compression type, and the presence of specific metadata files must exactly match what BambuStudio produces. `write_3mf()` copies everything verbatim from the source zip and only replaces `Metadata/plate_1.gcode` and `Metadata/plate_1.gcode.md5`.

### Cooldown: M190 S × 40, not M190 R

`M190 R{temp}` has a single ~90s firmware timeout and bailed out early (observed: stopped at 44°C instead of 25°C). `M190 S{temp}` repeated 40 times resets the timeout clock on each line. 40 × 90s = 60 min max. Once temp is reached, remaining lines complete instantly.

### Cooldown Z is dynamic, not hardcoded Z=1

Factorian hardcodes `G1 Z1` for the cooldown position. For tall prints this crashes the print top into the gantry/AMS as the bed rises. We use `push_z` (same as the Factorian push height formula: `max_z > 31 → max_z - 30`, else Z1) as the cooldown Z instead. This is safe for any print height.

### FarmLoop flex uses absolute Z, not relative to push height

The FarmLoop Stage 1 clip engages at a fixed physical position on the frame (**Z204** = 26 × 2mm jog clicks up from Z256 floor). The flex is `Z204 → Z224` (10 × 2mm jog clicks past engagement). This is independent of the print's max_z. Toolhead stays parked at X65 Y265 throughout — the clip is a mechanical device that acts on the bed regardless of toolhead position.

### GCODE structural markers the firmware requires

```
; EXECUTABLE_BLOCK_START
  ... motion init lines ...
; FEATURE: Custom
  ... machine start gcode ...
; MACHINE_START_GCODE_END
  ... print body ...
; FEATURE: Custom
; MACHINE_END_GCODE_START
  ... end gcode ...
; EXECUTABLE_BLOCK_END
```

`M73 P100 R0` must appear before `; EXECUTABLE_BLOCK_END`. MD5 must be uppercase hex, no newline. GCODE must use LF line endings. Missing any of these causes firmware error `0500-4003`.

### Post-processing scripts in BambuStudio are non-functional

BambuStudio's post-processing scripts field only runs on plain `.gcode` export, never on `.gcode.3mf`. All post-processing must happen externally via this script.

---

## Key Functions

| Function | Purpose |
|---|---|
| `parse_header(lines)` | Extracts `max_z_height`, `print_x_min/max`, temps from GCODE header |
| `find_max_z_from_toolpath(lines)` | Fallback: scans G1 Z moves for highest Z |
| `build_end_sequence(...)` | Generates the farm loop end GCODE block |
| `build_test_gcode(lines, ...)` | Builds test GCODE stripping at `M204 S10000` |
| `write_3mf(output, gcode, source)` | Repacks source zip replacing only gcode+MD5 |
| `read_input_3mf(path)` | Reads gcode lines and `plate_1.json` from a `.gcode.3mf` |
| `strip_end_gcode(lines)` | Finds and cuts at `; MACHINE_END_GCODE_START` |
| `push_x_from_plate_json(json)` | Extracts X centre from bbox in `plate_1.json` |

---

## Known Gaps / Future Work

- **Idempotency guard**: No check to prevent double-processing a file.
- **Exclusion zones for FarmLoop clip**: Must be set at slice time in BambuStudio.
- **MQTT auto-restart**: Queue management belongs in BambuBuddy, not here.
- **`G1 Y-3` HMS error**: AMS retract uses `G1 Y-3` (3mm past front bed limit, from Factorian's original). Triggers HMS 0300-0100-0003-0008 but does not abort the print. Kept to match Factorian exactly.

---

## Development Notes

- **Language**: Python 3, stdlib only (`zipfile`, `re`, `json`, `argparse`, `hashlib`, `pathlib`)
- **No third-party dependencies**
- **Single file**: `farm_loop.py`
- **Tested against**: BambuStudio 02.05.00.66, P1S firmware 01.08.x and 01.09.x
- **Platform**: Developed on Windows, deployed on Ubuntu 24.04 BambuBuddy host
