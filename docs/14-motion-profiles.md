# Motion-profile reference values

The motion-profile opcodes (`0x06` immediate, `0x0F` queued) carry per-axis acceleration / deceleration / velocity tables. The protocol does not prescribe specific values; the host chooses them. This page documents the reference values the stock host ships, so a reimplementation can boot with a working profile and tune from there.

## Speed-mode index

The host has **12 named speed modes** (indices 0..11) per axis. Mode 0 is the slowest (jog crawl, calibration); mode 11 is the highest configured speed. Each axis has its own 12-entry table because the kinematics differ (Z plunge is slow; XY gantry is fast; rotation is medium).

When the host issues `0x06 set_motion_profile`, it selects ONE row from its 6 × 12 table (the axis being configured) and packs the values into the 76-byte payload along with the speed percent.

## Effective velocity

The final velocity sent to the controller is:

```
velocity_out = profile_velocity[axis, speed_mode] * speed_pct / 100
```

where `speed_pct` is a value 1..100 passed as the speed-percent argument when the host emits `set_motion_profile`. For jog, the host always passes `100`. For job motion, the value comes from the host's UI / job settings.

**This is independent of parameter key 59 (`motor_subdivision`).** The motor-subdivision value scales baseline kinematic calculations elsewhere in the host but does NOT appear in the speed-percent multiplier.

## Reference tables (stock host values)

### Acceleration `int_6[axis, mode]` and deceleration `int_7[axis, mode]`

The stock host uses the same table for accel and decel (`int_6 == int_7`):

| Axis | Mode 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 |
|------|--------|---|---|---|---|---|---|---|---|---|----|----|
| 0    | 3      | 3 | 3 | 3 | 3 | 3 | 4 | 4 | 4 | 4 |  4 |  5 |
| 1    | 3      | 3 | 3 | 3 | 3 | 3 | 4 | 6 | 8 | 10| 18 | 20 |
| 2    | 3      | 4 | 5 | 3 | 3 | 3 | 4 | 5 | 6 | 7 | 10 | 12 |
| 3    | 3      | 3 | 3 | 3 | 4 | 4 | 5 | 5 | 6 | 6 |  7 |  7 |
| 4    | 3      | 4 | 5 | 3 | 3 | 3 | 3 | 4 | 5 | 6 |  8 | 10 |
| 5    | 3      | 3 | 3 | 3 | 4 | 4 | 5 | 5 | 6 | 6 |  7 |  7 |

Note that axis indices follow the protocol convention: 0 = peel, 1 = Z, 2 = X, 3 = A1, 4 = Y, 5 = A2.

### Velocity `int_8[axis, mode]`

| Axis | Mode 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 |
|------|--------|---|---|---|---|---|---|---|---|---|----|----|
| 0    | 1000   | 1500 | 2000 | 3000 | 4000 | 4500 | 5000 | 5000 | 5500 | 6000 |  8000 |  9000 |
| 1    | 150    |  500 | 1000 | 2000 | 3000 | 4000 | 5000 | 7000 | 9000 |11000 | 22000 | 22000 |
| 2    | 150    |  500 | 1500 | 2500 | 5000 | 9000 |12000 |15000 |17000 |20000 | 20000 | 20000 |
| 3    | 500    | 1000 | 2000 | 3000 | 4000 | 5000 | 7000 | 9000 |11000 |14000 | 20000 | 20000 |
| 4    | 150    |  500 | 1500 | 2500 | 5000 | 9000 |12000 |15000 |17000 |20000 | 20000 | 20000 |
| 5    | 500    | 1000 | 2000 | 3000 | 4000 | 5000 | 7000 | 9000 |11000 |14000 | 20000 | 20000 |

Units are controller-internal (not directly mm/s or steps/s without applying the per-axis scale factor; see [`04-axes.md`](04-axes.md) for the position encoding). For reimplementation work, treat the values as opaque "the stock host's reference" — tune up or down by ratio if the machine feels too slow / too jerky.

## Choosing a speed mode

The stock host's job loop typically uses mode **6-8** for transit moves and mode **0-2** for vision-step positioning. Mode 11 is reserved for cases where the operator explicitly asked for "fast jog" via the speed-select bits.

The 12-row design is a conservative tuning convention — operators rarely need that many discrete speeds. A reimplementation can collapse to 3-4 modes (slow / normal / fast / jog-only) without losing useful functionality, as long as the absolute values stay roughly within the range above.

## Frame layout of opcode 0x06

For reference (the spec already documents this; repeated here for convenience):

```
00: 06           opcode set_motion_profile_immediate
01: 00           sequence
02: AA           target axis (0..5)
03: 00           pad
04..27: u32×6    accel values (per-axis, only [target axis] is honoured)
28..51: u32×6    decel values
52..75: u32×6    velocity values
```

The speed-percent multiplier is applied by the **host** before packing — the controller receives already-scaled velocities. (Earlier audit notes mentioning a "speed percent argument" refer to the host-side `method_61` argument, NOT a field on the wire.)
