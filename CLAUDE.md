# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

`farm_loop.py` — a single-file Python 3 post-processor for Bambu P1S farm loop automation. It takes a BambuStudio-exported `.gcode.3mf`, strips the stock end GCODE, and injects a custom end sequence based on **FactorianDesigns Print Automation Custom Codes V2** (`Print_Automation_Costum_Codes_by_FactorianDesigns_Version2/`). See `FARM_LOOP_CONTEXT.md` for hardware context, architecture decisions, and hard-won lessons. **Read it before making changes.**

## Running

```bash
# Normal mode
python farm_loop.py input.gcode.3mf

# Test mode — validates push motion with a physical object on the bed
python farm_loop.py input.gcode.3mf --test

# Common options
python farm_loop.py input.gcode.3mf --cooldown-temp 25 --push-speed 300
python farm_loop.py input.gcode.3mf --test --push-x 128 --lane-offset 60
```

No dependencies beyond Python 3 stdlib. No build step, no install.

## Input files

The script only accepts `.gcode.3mf` files (BambuStudio plate export). Plain `.gcode` is not accepted by the P1S over the network and is not supported.

## Architecture

The script rewrites the GCODE inside the `.gcode.3mf` zip. The critical constraint is that the zip must be **repacked from the original source**, not generated from scratch — Bambu firmware is strict about internal file order, per-file compression types, and the presence of specific metadata files. See `write_3mf()`.

**Data flow:**
1. `read_input_3mf()` — extracts `Metadata/plate_1.gcode` lines and `Metadata/plate_1.json`
2. `parse_header()` — pulls `max_z_height`, print X bounds, temps from the GCODE header comments
3. `push_x_from_plate_json()` — falls back to bbox in `plate_1.json` for X centre
4. `strip_end_gcode()` — cuts at `; MACHINE_END_GCODE_START`
5. `build_end_sequence()` — generates the Factorian-based farm loop end block
6. `write_3mf()` — repacks, replacing only `plate_1.gcode` and `plate_1.gcode.md5`

**Test mode** (`build_test_gcode`): strips everything after `M204 S10000` in the machine start block (last safe init line before heatbed preheat), inserts G28 + 10s dwell + end sequence.

## End Sequence (FactorianDesigns V2)

1. Retract filament, raise Z slightly, move to safe/wipe pos (X65 Y265)
2. Fans off
3. AMS filament retract (`M620 S255` → `T255` → `M621 S255`)
4. Timelapse end (`M991 S0 P-1`)
5. Lower Z motor current, move bed to cooldown position (Z=1, toolhead is at rear wipe zone)
6. Aux + chamber fans on for active cooling
7. `M190 R{cooldown_temp}` — wait for bed to reach target temp
8. Fans off
9. **Z push height**: `max_z > 31mm → Z = max_z - 30`, else `Z = 1`
10. 3-lane push sweeps: center (`push_x`, F300 slow), right (`push_x + lane_offset`, F2000), left (`push_x - lane_offset`, F2000)
11. Reset feedrate/acc/speed level, lower motor current `M17 X0.8 Y0.8 Z0.5`, `M73 P100 R0`

## GCODE structural requirements

The firmware requires these exact markers in order:
```
; EXECUTABLE_BLOCK_START
; FEATURE: Custom
; MACHINE_START_GCODE_END
; FEATURE: Custom
; MACHINE_END_GCODE_START
; EXECUTABLE_BLOCK_END
```

`M73 P100 R0` must appear before `; EXECUTABLE_BLOCK_END`. MD5 must be uppercase hex, no newline. GCODE must use LF line endings (not CRLF). Missing any of these causes firmware error `0500-4003`.

## Key parameters

| Parameter | Default | Notes |
|---|---|---|
| `--cooldown-temp` | 25 | Bed °C before ejection |
| `--push-speed` | 300 | mm/min for center push (slow, Factorian default) |
| `--push-x` | auto | X centre; auto-detected from header or `plate_1.json` bbox |
| `--lane-offset` | 60 | mm left/right of `push_x` for side lanes |

## Reference material

- `Print_Automation_Costum_Codes_by_FactorianDesigns_Version2/P1 Codes/End_Code_Automatic_Pushoff_P1_V2.txt` — the end code this script is based on
- `Print_Automation_Costum_Codes_by_FactorianDesigns_Version2/P1 Codes/Start_Code_Short_Startup_P1.txt` — reference start code

## Known gaps

- No idempotency guard (double-processing not detected)
- Exclusion zone for FarmLoop clip must be set at slice time in BambuStudio, not here
- Queue/auto-restart belongs in BambuBuddy, not in this script
