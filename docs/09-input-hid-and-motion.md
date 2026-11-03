# 09 — Input, HID & Motion

Buttons, sticks, touch, gyro — and how ports remap them.

---

## 1. Buttons and sticks

```c
sceCtrlSetSamplingMode(SCE_CTRL_MODE_ANALOG_WIDE);   /* 0..255 per axis */

SceCtrlData pad;
memset(&pad, 0, sizeof(pad));
sceCtrlReadBufferPositive(0, &pad, 1);

if (pad.buttons & SCE_CTRL_CROSS) { /* ... */ }
float lx = pad.lx / 255.0f * 2.0f - 1.0f;
```

| Constant | Button |
|----------|--------|
| `SCE_CTRL_SELECT` / `SCE_CTRL_START` | Select / Start |
| `SCE_CTRL_UP/DOWN/LEFT/RIGHT` | D-pad |
| `SCE_CTRL_LTRIGGER` / `SCE_CTRL_RTRIGGER` | L / R |
| `SCE_CTRL_TRIANGLE/CIRCLE/CROSS/SQUARE` | face buttons |
| `SCE_CTRL_L3` / `SCE_CTRL_R3` | stick clicks |
| `SCE_CTRL_L2` / `SCE_CTRL_R2` ** | only on PSTV with a DualShock |

`SCE_CTRL_MODE_ANALOG_WIDE` gives the full 8-bit range; the default mode clips the
stick to a smaller effective range and makes aiming feel dead.

Ports commonly sample once per frame into a cached struct rather than calling
`sceCtrlReadBufferPositive` from everywhere.

## 2. Touch

```c
sceTouchSetSamplingState(SCE_TOUCH_PORT_FRONT, SCE_TOUCH_SAMPLING_STATE_START);
SceTouchData touch;
sceTouchPeek(SCE_TOUCH_PORT_FRONT, &touch, 1);
/* touch.reportNum points; touch.report[i].x/y are in 1920x1088 space */
```

Front panel coordinates are reported in a **1920×1088** space, not 960×544. Halve
them. `SCE_TOUCH_PORT_BACK` is the rear pad (and the only way to get L2/R2-style
input on a non-PSTV unit).

## 3. Motion

`sceMotionGetState` gives accelerometer + gyro + a fused orientation quaternion if
`sceMotionSensorStateEnable` has been called. Used for tilt-controls, camera
shake, and (rarely) as an input axis. Costs a sysmodule load.

## 4. Remapping — the pattern all three ports use

Keep a table from logical action to physical button, persist it, and sample through
it once per frame:

```
action           physical
---------------------------
confirm          CROSS
cancel           CIRCLE
attack           SQUARE
jump             TRIANGLE
block            LTRIGGER
sprint           RTRIGGER
grab             SELECT
pause            START
camera           RIGHT_STICK
move             LEFT_STICK
```

Melee's Vita mapping is a good default reference because it maps a GameCube pad with
missing buttons onto the Vita's layout without losing an action:

| Vita | GameCube |
|------|----------|
| Left stick | main stick |
| Right stick | C-stick |
| Cross / Circle | A / B |
| Square / Triangle | X / Y |
| L / R | L / R (full press only) |
| Select | Z (grab) |
| Start | Start |

The full-press-only note matters: GameCube had analogue triggers, the Vita does not.
Ports either quantise (press = full) or use a threshold on a stick axis.

## 5. Missing input is a design problem, not a mapping problem

`SCE_CTRL_L2`/`R2` exist only with a DualShock on PSTV. If the original game needs
more simultaneous inputs than the Vita provides, the answer is contextual mapping
(modifier + button), not a clever chord — chords are unplayable in combat.

## 6. Sources

- `Sdk/include/psp2/ctrl.h`, `touch.h`, `motion.h`
- `Other Emulators/play-master/Source/ui_vita/main.cpp` (pad → emulated controller)
- `Other Emulators/PCSX_AwO-master/frontend/vita/input_mapping.h`
- `Other Ports/melee-Vita-vita-port/platforms/vita/game/pad.c` + `pad_mapping.h`
