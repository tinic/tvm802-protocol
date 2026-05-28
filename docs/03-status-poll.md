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

| Bit  | Mask     | Meaning                                |
|------|----------|----------------------------------------|
| 0    | `0x0001` | System moving (any axis in motion)     |
| 1    | `0x0002` | Job running                            |
| 2    | `0x0004` | Home cycle active                      |
| 4    | `0x0010` | Limit hit                              |
| 5-15 | various  | Vendor-specific; treat as opaque       |

The cleanest "is anything moving?" test is the per-axis `axis_state[i]` bytes; the `status_word` bit 0 is essentially `OR(axis_state)`.

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
