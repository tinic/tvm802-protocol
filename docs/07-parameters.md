# Parameter store

The controller hosts a keyed configuration store of ~601 signed 32-bit integers. Three opcodes operate on it:

- `0x01 read_param` — read one parameter.
- `0x02 write_param` — write one parameter (uncommitted).
- `0x03 commit_params` — latch pending writes.

A reimplementation that only **reads** parameters is safe to run alongside the live host; writes are not — see [01-transport.md](01-transport.md) for the multi-client guarantee.

## Structure

The key space is sparse. There are two distinct regions:

| Region        | Keys     | Shape                          | Contents                                     |
|---------------|----------|--------------------------------|----------------------------------------------|
| Scalars       | 0..61    | Named individually             | Machine ID, IP, home, vision pos, scales, thresholds, Z heights, calibration, feature flags |
| Table A       | 85..99   | 15 entries, flat               | Identity / payload (likely firmware ident)   |
| Table B       | 100..279 | 60 entries × 3 keys (triples)  | Tape-feeder slot configuration               |
| Table C       | 300..359 | 30 entries × 2 keys (pairs)    | Down-vision fiducial set 1                   |
| Table D       | 360..419 | 30 entries × 2 keys (pairs)    | Down-vision fiducial set 2                   |
| Table E       | 420..449 | 30 entries × 1 key             | Tray-stack metadata                          |
| Table F       | 450..479 | 30 entries × 1 key             | Advanced vision / component config           |

Keys outside these ranges are mostly reserved or unused, with one exception:

| Range     | Status                                                                           |
|-----------|----------------------------------------------------------------------------------|
| 62..84    | Unused; reads as `-1`.                                                           |
| 280..299  | Gap between Table B (feeders) and Table C (fiducials); reads as `-1`.            |
| 480..499  | Gap between Table F and the persistent region; reads as `-1`.                    |
| **500..540** | **In use.** Key 500 is the host-managed `machine_homed` flag (above). Keys 501..540 contain controller-side persistent / factory values that the stock host does NOT read but the controller writes (likely chip IDs, factory limits, diagnostic counters). Safe to read; **do not write**. |
| 541..600  | Unused; reads as `-1`.                                                           |

## Frames

### Read (opcode 0x01, 12-byte request / 12-byte response)

```
Req:
  00: 01           opcode
  01: 00           sequence
  02: 00           aux
  03: 00           pad
  04: KK KK KK KK  key, u32 LE
  08: 00 00 00 00  pad
Rsp:
  00: 01           opcode_echo
  08: VV VV VV VV  value, s32 LE
```

### Write (opcode 0x02, 12-byte request)

```
00: 02           opcode
04: KK KK KK KK  key
08: VV VV VV VV  value, s32 LE
```

### Commit (opcode 0x03, 4-byte request)

```
00: 03           opcode
```

A write burst MUST end with one commit. Writes without a commit are silently discarded by the controller on the next reset.

## Unit conventions

Every parameter value is a signed 32-bit integer on the wire. The decode rule is per-key:

| Unit               | Decode rule                              | Where used                          |
|--------------------|------------------------------------------|-------------------------------------|
| `raw`              | `value`                                  | Serial, IP, feature flags           |
| `micron`           | `mm = value / 1000`                      | Positions, Z heights, offsets       |
| `micropx_per_mm`   | `mm_per_px = value / 10000`              | Camera scales                       |
| `centi`            | `real = value / 100`                     | Deviation thresholds                |
| `millis`           | `ms = value`                             | Settle delays, settle waits         |
| `bool`             | `flag = (value != 0)`                    | Single-bit feature flags            |
| `flag_byte_8`      | bits 0..7 are 8 separate booleans        | Key 23                              |
| `packed_ipv4_le`   | bytes 0..3 → IPv4 octets                 | Key 2 (controller IP)               |
| `packed_u16x2`     | `hi = (value >> 16) & 0xFFFF, lo = value & 0xFFFF` | Vacuum thresholds, vacuum scales |
| `packed_i16x2`     | same as above but signed                 | Camera mirror flags                 |
| `packed_byte_x4`   | 4 bytes MSB-first                        | Tables A and F                      |
| `u16`              | `value & 0xFFFF`                         | Port, machine-serial low half       |

## Scalars (keys 0-61)

Display labels in Chinese are taken from the host's ParamEdit dialog and preserved as the user would see them.

| Key | ID                          | Unit              | Label (CN)            | Notes                                                                  |
|----:|-----------------------------|-------------------|-----------------------|------------------------------------------------------------------------|
| 0   | `machine_serial_hi`         | raw               | MAC地址               | Machine serial / MAC, high 32 bits.                                    |
| 1   | `machine_serial_lo`         | u16               | MAC地址               | Machine serial / MAC, low 16 bits.                                     |
| 2   | `controller_ip`             | packed_ipv4_le    |                       | `0x0800A8C0` = 192.168.0.8 (factory default).                          |
| 3   | `controller_port`           | u16               |                       | Default 701.                                                           |
| 4   | `home_x`                    | micron            | 原点坐标 (X)          | Home position X (machine zero).                                        |
| 5   | `home_y`                    | micron            | 原点坐标 (Y)          | Home position Y.                                                       |
| 6   | `n1_upcam_pos_x`            | micron            | 相机坐标1 (X)         | Head-1 centred over up-vision camera, X.                               |
| 7   | `n1_upcam_pos_y`            | micron            | 相机坐标1 (Y)         | Head-1 over up-camera, Y.                                              |
| 8   | `cam_offset_x`              | micron            |                       | Camera-to-machine reference offset, X.                                 |
| 9   | `cam_offset_y`              | micron            |                       | Camera reference offset, Y.                                            |
| 10  | `prick_offset_x`            | micron            |                       | Prick-pin offset from head reference, X.                               |
| 11  | `prick_offset_y`            | micron            |                       | Prick-pin offset, Y.                                                   |
| 12  | `discard_x`                 | micron            | 弃料坐标 (X)          | Reject / discard bin position, X.                                      |
| 13  | `discard_y`                 | micron            | 弃料坐标 (Y)          | Discard bin Y.                                                         |
| 14  | `nozzle_z_pcb`              | micron            |                       | Nozzle plunge Z at PCB.                                                |
| 15  | `nozzle_z_feeder`           | micron            |                       | Nozzle plunge Z at tape-feeder pickup.                                 |
| 16  | `nozzle_z_front_stack`      | micron            |                       | Nozzle plunge Z at front-tray pickup.                                  |
| 17  | `nozzle_z_discard`          | micron            |                       | Nozzle plunge Z at discard drop.                                       |
| 18  | `prick_z`                   | micron            |                       | Prick-pin plunge depth.                                                |
| 19  | `feature_flag_misc`         | bool              |                       | Generic feature flag (machine-specific).                               |
| 20  | `upcam_scale_x`             | micropx_per_mm    | 相机比例 (X)          | Up-vision X scale.                                                     |
| 21  | `upcam_scale_y`             | micropx_per_mm    | 相机比例 (Y)          | Up-vision Y scale.                                                     |
| 22  | `vision_z`                  | micron            |                       | Up-vision focal Z; often 0.                                            |
| 23  | `feature_flags_8`           | flag_byte_8       |                       | 8 packed feature flags; see table below.                               |
| 24  | `nozzle_vacuum_thresh`      | raw               | 吸嘴阈值              | Nozzle-empty vacuum threshold (low).                                   |
| 25  | `comp_vacuum_thresh`        | raw               | 元件阈值              | Component-present vacuum threshold (high).                             |
| 26  | `nozzle_vision_z`           | micron            |                       | Nozzle Z at vision pose; often 0.                                      |
| 27  | `nozzle_offset_x`           | micron            |                       | Nozzle offset relative to head reference, X.                           |
| 28  | `nozzle_offset_y`           | micron            |                       | Nozzle offset, Y.                                                      |
| 29  | `n1_feeder_settle_ms`       | millis            |                       | Nozzle-1 settle delay at feeder pickup.                                |
| 30  | `pcb_settle_ms_a`           | millis            |                       | Settle delay after nozzle plunge at PCB (variant A).                   |
| 31  | `pcb_settle_ms_b`           | millis            |                       | Settle delay after nozzle plunge at PCB (variant B).                   |
| 32  | `n2_feeder_settle_ms`       | millis            |                       | Nozzle-2 settle delay at feeder pickup.                                |
| 33  | `auto_sort_flag`            | bool              |                       | Auto-sort placement order. `-1` (all bits) common in backups.          |
| 34  | `downcam_scale_x`           | micropx_per_mm    | 下视相机比例 (X)      | Down-vision X scale.                                                   |
| 35  | `downcam_scale_y`           | micropx_per_mm    | 下视相机比例 (Y)      | Down-vision Y scale.                                                   |
| 36  | `up_vision_angle`           | micron            |                       | Up-vision angle-calibration constant.                                  |
| 37  | `prick_retract`             | micron            | 拨针回退              | Prick-pin retract distance after feed.                                 |
| 38  | `nozzle_cal_x`              | micron            |                       | Nozzle calibration X.                                                  |
| 39  | `nozzle_cal_y`              | micron            |                       | Nozzle calibration Y.                                                  |
| 40  | `nozzle_cal_z`              | micron            |                       | Nozzle calibration Z primary.                                          |
| 41  | `nozzle_cal_z2`             | micron            |                       | Nozzle calibration Z secondary.                                        |
| 42  | `mark_vision_offset_x`      | micron            | 下视中心 (X)          | Down-camera fiducial centre offset, X.                                 |
| 43  | `mark_vision_offset_y`      | micron            | 下视中心 (Y)          | Down-camera fiducial centre offset, Y.                                 |
| 44  | `prick_offset_x_alt`        | micron            |                       | Prick-pin offset (secondary set), X.                                   |
| 45  | `prick_offset_y_alt`        | centi             |                       | Prick-pin offset (secondary), Y, scaled × 100.                         |
| 46  | `prick_pos_x`               | micron            |                       | Prick-pin assembly mounting position, X.                               |
| 47  | `prick_pos_y`               | micron            |                       | Prick-pin assembly mounting position, Y.                               |
| 48  | `up_cam_settle_ms`          | millis            |                       | Up-camera settle delay after camera switch.                            |
| 50  | `controller_scale_x`        | micropx_per_mm    |                       | Controller-internal X scale (~1.0 calibration factor).                 |
| 51  | `controller_scale_y`        | micropx_per_mm    |                       | Controller-internal Y scale.                                           |
| 52  | `dev_thresh_xy`             | centi             |                       | XY-deviation threshold for placement correction warnings.              |
| 53  | `dev_thresh_angle`          | centi             |                       | Angle-deviation threshold (degrees × 100).                             |
| 54  | `vacuum_thresh_pack_a`      | packed_u16x2      |                       | Vacuum thresholds, channels 1 & 2.                                     |
| 55  | `vacuum_thresh_pack_b`      | packed_u16x2      |                       | Vacuum thresholds, channels 3 & 4.                                     |
| 56  | `z_home_offset`             | micron            |                       | Z home offset; write-only, host writes during homing.                  |
| 57  | `vacuum_scale_pack`         | packed_u16x2      |                       | Vacuum analog scaling: `(hi/100, lo/100)` multiply `vacuum_*_raw`.     |
| 58  | `camera_mirror_pack`        | packed_i16x2      |                       | `(up_mirror << 16) | (down_mirror & 0xFFFF)`. MVision-DLL flip codes.  |
| 59  | `motor_subdivision`         | raw               | 电机细分              | Motor microstep subdivision (UI shows raw / 8). Also acts as a velocity multiplier in motion equations — changing it rescales jog and move speeds. |
| 60  | `n2_upcam_pos_x`            | micron            | 相机坐标2 (X)         | Head-2 centred over up-vision camera, X.                               |
| 61  | `n2_upcam_pos_y`            | micron            | 相机坐标2 (Y)         | Head-2 over up-camera, Y.                                              |
| 500 | `machine_homed`             | bool              |                       | Persistent "machine has been homed" flag. Set true after GoHome completes; read as a precondition gate by calibration / debug dialogs. Persists in the controller across host restarts. |

### Feature-flag byte (key 23)

`value & 0xFF` is bit-packed:

| Bit | ID                          | Meaning                                              |
|-----|-----------------------------|------------------------------------------------------|
| 0   | `use_pressure_check`        | Enable vacuum-pressure pickup confirmation.          |
| 1   | `use_auto_sort`             | Re-order placements by proximity.                    |
| 2   | `use_auto_return_home`      | Return to home after job complete.                   |
| 3   | `left_stack_positioning`    | LEFT-bank feeder positioning mode.                   |
| 4   | `back_stack_positioning`    | BACK-bank feeder positioning mode.                   |
| 5-7 | reserved                    | Reserved / undocumented.                             |

### Vacuum threshold pairs (keys 54, 55)

Each key packs two `u16` thresholds:

| Field         | Channel | Notes                                  |
|---------------|---------|----------------------------------------|
| `key 54 hi`   | 1       | Head-1 vacuum threshold A.             |
| `key 54 lo`   | 2       | Head-2 vacuum threshold A.             |
| `key 55 hi`   | 3       | Head-1 vacuum threshold B (aux).       |
| `key 55 lo`   | 4       | Head-2 vacuum threshold B (aux).       |

## Table A — machine identity / payload (keys 85-99)

15 entries, flat. Each key is four bytes (packed_byte_x4). In a typical backup all 15 keys hold `0x3C3C3C3C` (default fill). The purpose is most likely firmware identity, model code, or vendor payload — not fully decoded.

## Table B — tape-feeder slot configuration (keys 100-279)

**60 feeder slots**, each occupying three sequential keys:

| Offset within slot | Key (slot N)   | Field           | Decode                                          |
|--------------------|----------------|-----------------|-------------------------------------------------|
| 0                  | 100 + 3·N      | `pos_x`         | `mm = value / 1000`                             |
| 1                  | 101 + 3·N      | `pos_y`         | `mm = value / 1000`                             |
| 2                  | 102 + 3·N      | `type_id_pack`  | `type = (value >> 16) & 0xFFFF; id = value & 0xFFFF` |

The feeder `type` is an enum that maps to a component-height LUT:

| Type | Height (mm) |
|------|-------------|
| 8    | 3.5         |
| 12   | 5.5         |
| 16   | 7.5         |
| 24   | 11.0        |

Slots **0-29** sit on the **LEFT bank** (X ≈ -45 mm, Y varies — feed drags on **X**).
Slots **30-59** sit on the **BACK bank** (Y ≈ 367 mm, X varies — feed drags on **Y**).

Example, slot 0:

```
key 100 = -45230   →  pos_x = -45.230 mm
key 101 =  -4690   →  pos_y =  -4.690 mm
key 102 = 524292   →  type  = 8, id = 4, height = 3.5 mm
```

## Table C — fiducial reference set 1 (keys 300-359)

**30 board-fiducial positions** for the down-camera. Each entry has two keys:

| Offset | Key (slot N) | Field   | Decode                  |
|--------|--------------|---------|-------------------------|
| 0      | 300 + 2·N    | `pos_x` | `mm = value / 1000`     |
| 1      | 301 + 2·N    | `pos_y` | `mm = value / 1000`     |

Sparsely used in practice — most slots read 0.

## Table D — fiducial reference set 2 (keys 360-419)

Identical structure to Table C; same 30 × 2 layout. Purpose mirrors Table C and is most likely an alternate layout or bottom-side calibration set.

## Table E — tray-stack metadata (keys 420-449)

**30 single-key entries**, each packing two `u16` values:

| Field             | Decode                            |
|-------------------|-----------------------------------|
| `stack_type`      | `(value >> 16) & 0xFFFF`          |
| `instance_id`     | `value & 0xFFFF`                  |

Used for the front tray-stack feeders (orientation, count, feed direction).

## Table F — advanced vision / component config (keys 450-479)

**30 single-key entries**, each packed as four bytes (MSB first). Each byte represents a sub-value × 100. Written by the newer parameter-edit path but not always read by older firmware. Purpose not fully decoded; confidence: low.

## Backup and restore

**Before** any write, back up the full key space:

```
for key in 0..600:
    value = read_param(key)
    log(key, value)
```

On a typical TVM802B about 476 keys are non-zero. Restore writes each non-zero key back via `write_param` and ends with a final `commit_params`. Always test restore on a non-critical key (e.g. a feature flag at known value) before trusting a full restore.

Backup files in the wild can be in any text format — most reimplementations use `key=value` lines, one per key, which round-trips cleanly with the read/write opcodes documented here.
