# Status poll (opcode 0x00)

The status poll is the most important frame in the protocol. It is issued every host UI tick (~50 ms) and carries the entire live machine state in 47 bytes.

There is no separate "live position" or "live IO" query. Everything is in here.

## Request

```
Offset  Size  Field         Value
------  ----  ------------  ----------------
0       1     opcode        0x00
1       1     sequence      0x00
2       1     aux           0x00
3       1     pad           0x00
```

Total request size: **4 bytes**.

## Response

Total response size: **47 bytes**.

```
Offset  Size  Field                Decode
------  ----  -------------------  ----------------------------------------
0       1     opcode_echo          Always 0x00
1       1     sequence
2       1     queue_free           Free slots in controller FIFO (~7 max)
3       1     reserved
4       2     status_word          Bit-packed; see status_word below
6       2     reserved
8       1     input_byte_0         See digital_inputs in spec/protocol.yaml
9       1     input_byte_1
10      1     input_byte_2
11      1     input_byte_3
12      4     axis_position[0]     Peel, int32 LE
16      4     axis_position[1]     Z
20      4     axis_position[2]     X
24      4     axis_position[3]     A1
28      4     axis_position[4]     Y
32      4     axis_position[5]     A2
36      1     axis_state[0]        0 = idle, nonzero = busy
37      1     axis_state[1]
38      1     axis_state[2]
39      1     axis_state[3]
40      1     axis_state[4]
41      1     axis_state[5]
42      2     vacuum_1_raw         u16 LE, head-1 vacuum-pressure analog
44      2     vacuum_2_raw         u16 LE, head-2 vacuum-pressure analog
46      1     event_code           Controller event/error code; 0 = none
```

## Decoding the axis positions

The axis-position raw value is a signed int32 little-endian, encoding millimetres scaled by a per-axis constant:

```
mm = (raw / 1000) / scales[axis]
```

with `scales = [1.0, 1.0, 32.8084, 4.4444, 32.8084, 4.4444]`.

The `*1000` factor is the controller's internal micron representation. The per-axis scale is the steps-per-mm (linear) or steps-per-degree (rotary) of the leadscrew / pulley / gear. For example:

- Axis 2 (X), raw `400000000` → `400000 / 32.8084` ≈ **12193 mm**? No — read more carefully: `raw=400000000` ÷ 1000 = `400000` micron-scaled steps, ÷ 32.8084 = **12193 µm** = 12.193 mm. The double-`/1000` is built into the scale: scale 32.8084 means *steps per mm* AND positions arrive in 1/1000ths of a "scaled millimetre".

For most reimplementations the right way is:

```python
mm = raw / 1000.0 / scales[axis]
```

and to encode the inverse for a move target:

```python
raw = round(mm * scales[axis] * 1000)
# but the host uses round(mm * scales[axis]) * 1000 — a 1-µm quantisation step.
```

## `status_word` bits

Best-effort mapping based on the host's individual bit-getter call sites. High-confidence bits are those the host actively gates decisions on; medium/low confidence are surfaced in UI or aggregated.

| Bit  | Mask     | Confidence | Meaning                                                                  |
|------|----------|------------|--------------------------------------------------------------------------|
| 0    | `0x0001` | **High**   | E-stop latched (hardware-side estop input).                              |
| 1    | `0x0002` | Medium     | Vacuum-sensor fault on head 2.                                           |
| 2    | `0x0004` | Medium     | Vacuum-sensor fault on head 1.                                           |
| 3    | `0x0008` | Low        | Reserved.                                                                |
| 4    | `0x0010` | Low        | Reserved (older docs called this "limit hit"; not confirmed).            |
| 5    | `0x0020` | Low        | Aggregated limit-latch (component of method_17's `& 0x408` aggregate).   |
| 6    | `0x0040` | Low        | Aggregated limit-latch.                                                  |
| 7    | `0x0080` | Low        | Aggregated limit-latch.                                                  |
| 8    | `0x0100` | Low        | Aggregated limit-latch.                                                  |
| 9    | `0x0200` | **High**   | Motion in progress (any axis moving). Equivalent to `OR(axis_state[*])`. |
| 10   | `0x0400` | Medium     | Pump active / soft-limit aggregate component.                            |
| 11   | `0x0800` | **High**   | Program ready / idle (host gates "start job" on this).                   |
| 12   | `0x1000` | Low        | Reserved.                                                                |
| 13   | `0x2000` | Medium     | Prick-pin home (overlaps input bit b22).                                 |
| 14   | `0x4000` | Low        | Reserved.                                                                |
| 15   | `0x8000` | Low        | Reserved.                                                                |

Reimplementation rule of thumb:

- **Gate logic on bits 0, 9, 11.** E-stop / motion / ready — these are reliable and the host trusts them.
- **Surface in UI bits 1, 2, 10, 13.** Useful diagnostics; do not gate behaviour on them without further validation.
- **Treat the rest as reserved.** Log them when nonzero; don't act on them.

The cleanest "is anything moving?" test remains the per-axis `axis_state[i]` bytes; `status_word` bit 9 essentially mirrors them.

## `event_code` (byte 46)

The stock host checks for exactly two non-zero event codes:

| Code | ID                  | Severity         | Meaning                                                          |
|------|---------------------|------------------|------------------------------------------------------------------|
| 0    | `none`              | nominal          | No event.                                                        |
| 1    | `cmd_order_error`   | error — e-stop   | Command-order violation (move issued in wrong sequence / state). |
| 2    | `limit_signal_error`| error — e-stop   | Motion commanded toward an already-asserted limit switch.        |

On either non-zero code, the stock host:

1. Asserts output bit `0x800` (buzzer).
2. Shows a dialog (resource strings `"cmdOrderError"` / `"limitSignalError"`).
3. Runs an emergency-stop sequence.

### Recovery sequence (after operator dismisses the dialog)

```
0x14 output_bit_on_immediate  bit = 0x0800   # buzzer (already on; idempotent)
0x09 stop                     axis_mask = 0x3F   # all-axis stop
```

The stock host does NOT re-issue opcode `0x05` (controller_init) or `0x0B cmd=0` (program_control clear) on this path. The motion FIFO is drained host-side; the controller's state recovers via the stop alone. After this, the host resumes its normal status-poll cadence. The operator must restart the job manually.

A reimplementation should treat any unknown non-zero `event_code` the same way as 1 / 2: stop, surface the raw code, log for diagnosis. The controller may report other codes in different firmware revisions or fault modes that the stock host doesn't recognise.

## Vacuum-analog scaling and host behaviour

The two vacuum readings (`vacuum_1_raw`, `vacuum_2_raw`, bytes 42-45) are raw u16 values from the controller's ADC. Parameter `57` carries per-channel scale factors (`hi / 100`, `lo / 100`) that *could* convert the raw values into a real unit (kPa, % of vacuum, etc).

**The stock host reads the scale factors but does not apply them.** Pickup confirmation uses the raw u16 directly:

```python
delta = baseline_raw - vacuum_raw
if delta < part_threshold_raw:
    pickup_failed
```

A reimplementation can match this behaviour (raw thresholds, baseline calibrated per machine) or apply the scale factors for human-friendly logging. Either approach is fine — the protocol exposes both halves.

## Vacuum analog channels

Bytes 42-45 are the two head vacuum-pressure analog readings, raw 16-bit. They are NOT bits or boolean pickup flags — pickup confirmation requires comparing the pressure delta against a per-part threshold:

```
delta = baseline_pressure - vacuum_raw
if delta < threshold: pickup_failed
```

The baseline and threshold are part-specific and live in the host's job data, not in the controller. The wire only gives you the raw reading.

## Polling cadence

The stock host polls at ~20 Hz (50 ms tick). For a reimplementation:

- **20 Hz** is plenty for the FSM-style placement loop.
- Faster polling (100 Hz) is fine but yields no extra information; the controller's internal update rate caps the useful rate.
- Slower than 10 Hz will make jog feel sluggish and may miss short input pulses (e.g. brief button taps).
