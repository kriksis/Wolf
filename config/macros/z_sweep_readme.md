# Z_SWEEP - Z Axis Resonance Speed Finder

Part of the STōN-WoLF tuning macro suite.

---

## The Problem

Klipper gives you one Z speed setting and assumes it is correct. It is not - it is a guess. Every machine has resonant frequencies baked into its frame, lead screw, and linear rail combination. Run Z at the wrong speed and you get audible noise, micro-vibrations at the start of every print, and potential first-layer artifacts as the toolhead settles. Nobody tells you what the right speed is because it depends on your specific hardware, your mounting, your frame rigidity, and a dozen other variables nobody can predict from a spec sheet.

The standard advice is "try different speeds until it sounds right." That means manually editing config, restarting firmware, homing, listening, repeating - for every candidate speed, one at a time. It takes forever and you will still miss speeds between your test points.

---

## The Fix

Z_SWEEP runs your Z axis through a user-defined speed range automatically, stepping in increments as fine as 0.1mm/s. Every move at every speed is at the exact test feedrate - no hidden fast returns, no cheating. The console announces each speed as it runs so you always know what you are listening to. You put a hand on the frame, watch the console scroll, and note what sounds and feels smooth. One run. Done.

---

## Installation

**1. Add the macro to any `.cfg` file Klipper loads.**

The simplest approach - save it as its own file and include it in `printer.cfg`:

```ini
[include Z_SWEEP.cfg]
```

If you already have a macro suite (like the STōN-WoLF `ston_macros.cfg`), just paste the macro block in there instead. No separate file needed.

**2. Add to your existing `[gcode_macro _USER_VARIABLES]` block:**

```ini
variable_z_sweep_reps:   3     ; bounces per speed
variable_z_sweep_dwell:  500   ; ms between moves
```

**2. Add the macro to `ston_macros.cfg`:**

```ini
[gcode_macro Z_SWEEP]
description: Z resonance sweep - MONITOR FIRST RUN, FINGER ON STOP! Find smoothest Z speed.
gcode:
  {% set sv     = printer["gcode_macro _USER_VARIABLES"] %}
  {% set start  = params.START|default(5.0)|float %}
  {% set end    = params.END|default(15.0)|float %}
  {% set step   = params.STEP|default(0.5)|float %}
  {% set z_low  = params.Z_LOW|default(20)|float %}
  {% set z_high = params.Z_HIGH|default(60)|float %}
  {% set reps   = sv.z_sweep_reps|int %}
  {% set dwell  = sv.z_sweep_dwell|int %}
  {% set start  = [start, 0.5]|max %}
  {% set start  = [start, 30.0]|min %}
  {% set end    = [end, 0.5]|max %}
  {% set end    = [end, 30.0]|min %}
  {% set step   = [step, 0.1]|max %}
  {% set steps  = ((end - start) / step)|round(0, 'ceil')|int + 1 %}
  {% if 'xyz' not in printer.toolhead.homed_axes %}
    M118 Homing all axes...
    G28
  {% elif 'z' not in printer.toolhead.homed_axes %}
    M118 Homing Z...
    G28 Z
  {% endif %}
  SAVE_GCODE_STATE NAME=z_sweep
  G90
  G1 Z{z_low} F600
  G4 P1000
  {% for i in range(steps) %}
    {% set mms = [[start + i * step, 30.0]|min, 0.5]|max %}
    {% set mms = mms|round(2) %}
    {% set f   = (mms * 60)|round(1) %}
    M118 --- {mms}mm/s (F{f}) x{reps} ---
    {% for r in range(reps + 1) %}
      G1 Z{ z_high if r % 2 == 0 else z_low } F{f}
      G4 P{dwell}
    {% endfor %}
  {% endfor %}
  M118 --- sweep done ---
  RESTORE_GCODE_STATE NAME=z_sweep
```

**3. `FIRMWARE_RESTART`**

---

## How to Use

Click **Z_SWEEP** in the Mainsail macros panel. A dialog appears with 5 fields:

| Field | Default | Description |
|---|---|---|
| `START` | 5.0 | Sweep start speed mm/s |
| `END` | 15.0 | Sweep end speed mm/s |
| `STEP` | 0.5 | Speed increment mm/s |
| `Z_LOW` | 20 | Lower travel position mm |
| `Z_HIGH` | 60 | Upper travel position mm |

Adjust or leave defaults and hit **Run**. The macro homes if needed, moves to `Z_LOW`, then steps through every speed. Console output:

```
--- 5.0mm/s (F300.0) x3 ---
--- 5.5mm/s (F330.0) x3 ---
--- 6.0mm/s (F360.0) x3 ---
--- sweep done ---
```

Hand on frame. Buzz or rattle at a speed = bad. Smooth and quiet = good. Write down the mm/s value.

---

## Applying Your Result

Open `printer.cfg` and find your `[stepper_z]` section. Update:

```ini
[stepper_z]
homing_speed: 16.2
second_homing_speed: 2.0
homing_retract_dist: 5
homing_retract_speed: 16.2
```

If you use `[homing_override]`, the speed is set inside that block as an `F` value. Multiply your target mm/s by 60 to get mm/min. 16.2mm/s = F972.

After saving, `FIRMWARE_RESTART` and re-home to confirm.

---

## Probe Compatibility

Probe agnostic. Works with Beacon, Smart Effector, BLTouch, CR Touch, plain endstop, or anything else. The macro never calls the probe directly - `G28 Z` uses whatever homing configuration you already have. No changes needed regardless of probe type.

---

## IMPORTANT - Monitor Your Machine During First Use

**Stay at the machine and watch it run the entire first time. Finger on the stop button.**

If your `Z_LOW` or `Z_HIGH` values are wrong, if the machine is not properly homed, if there is a config error, or if something is physically loose - it will show up as a problem during the sweep. You need to be there to hit emergency stop if anything looks wrong.

Once you have run it once and confirmed it behaves correctly, subsequent runs are routine. The first time - watch it.

---

## Notes

- `Z_LOW` and `Z_HIGH` define the travel range for the test. 40mm of travel is plenty to excite resonance. Keep `Z_LOW` at least 10mm above your bed surface.
- `reps` and `dwell` in `_USER_VARIABLES` are background settings. Defaults work for most users.
- Speed range is clamped to 0.5–30mm/s. Klipper will error if you exceed `max_z_velocity` in your config regardless.
- Safe to stop mid-run with emergency stop or cancel. `RESTORE_GCODE_STATE` fires on clean completion only.
- This macro does not write any result to your config. You make the change manually in `printer.cfg` after testing.

---

*STōN-WoLF - Built for people who actually use their machines.*
