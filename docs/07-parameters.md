# Parameter store

The controller hosts a keyed configuration store of ~601 int32 parameters. Three opcodes operate on it:

- `0x01 read_param` — read one parameter.
- `0x02 write_param` — write one parameter (uncommitted).
- `0x03 commit_params` — latch pending writes.

A reimplementation that only **reads** parameters is safe to run alongside the live host; writes are not.

## Frames

### Read

Request (12 bytes):

```
00: 01           opcode = read_param
01: 00           sequence
02: 00           aux
03: 00           pad
04: KK KK KK KK  key, u32 LE
08: 00 00 00 00  pad
```

Response (12 bytes):

```
00: 01           opcode_echo
01..07: ...
08: VV VV VV VV  value, s32 LE
```

### Write

Request (12 bytes):

```
00: 02           opcode = write_param
01..03: 00 00 00
04: KK KK KK KK  key
08: VV VV VV VV  value, s32 LE
```

Writes are buffered on the controller side. They take effect only after the next `commit_params`.

### Commit

```
00: 03           opcode = commit_params
01..03: 00 00 00
```

A write burst always ends with one commit. This is also the safe place to reload host caches of any param that was just written.

## Unit conventions

All parameter values are 32-bit signed integers. The unit depends on the key:

| Convention             | Decode                | Example keys           |
|------------------------|-----------------------|------------------------|
| `micron` (× 1000 mm)   | `mm  = raw / 1000`    | home, vision positions |
| `micropx_per_mm`       | `mm/px = raw / 10000` | camera scales          |
| `packed_ipv4_le`       | `b0.b1.b2.b3`         | key 2 (controller IP)  |
| `u16`                  | `raw & 0xFFFF`        | key 3 (port)           |
| `bool`                 | `raw != 0`            | feature flags          |
| `packed_i16x2`         | `hi = raw>>16, lo = raw & 0xFFFF` (signed) | key 58 (camera mirror) |

## Known keys

This list is the verified subset; the full key space is 0..600.

| Key | ID                      | Unit                |
|-----|-------------------------|---------------------|
| 2   | `controller_ip`         | packed_ipv4_le      |
| 3   | `controller_port`       | u16                 |
| 4   | `home_x`                | micron              |
| 5   | `home_y`                | micron              |
| 6   | `n1_upcam_pos_x`        | micron              |
| 7   | `n1_upcam_pos_y`        | micron              |
| 12  | `discard_x`             | micron              |
| 13  | `discard_y`             | micron              |
| 20  | `upcam_scale_x`         | micropx_per_mm      |
| 21  | `upcam_scale_y`         | micropx_per_mm      |
| 22  | `vision_z`              | micron              |
| 26  | `nozzle_vision_z`       | micron              |
| 34  | `downcam_scale_x`       | micropx_per_mm      |
| 35  | `downcam_scale_y`       | micropx_per_mm      |
| 45  | `feature_flag_v3_09`    | bool                |
| 56  | `z_home_offset`         | micron              |
| 58  | `camera_mirror_pack`    | packed_i16x2        |
| 60  | `n2_upcam_pos_x`        | micron              |
| 61  | `n2_upcam_pos_y`        | micron              |

A "dump everything" script reads keys 0..600 sequentially; on a typical TVM802B about 476 keys are non-zero. Many of the remaining keys carry feeder configuration, vision parameters (range, mark sizes, thresholds), and miscellaneous tuning constants. Future revisions of this spec will fill more in as they're verified.

## Backups

Before any write — back up. The simplest backup is:

```
for key in 0..600:
    value = read_param(key)
    log(key, value)
```

The stock host has no concept of "restore from backup"; you do this by writing each key back via `write_param` + a final `commit_params`. Test the round-trip on a single non-critical key (e.g. a scale) before trusting the full restore.
