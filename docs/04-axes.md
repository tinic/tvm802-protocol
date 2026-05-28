# Axis model

The protocol exposes **6 axes**, indexed 0..5. Several map to non-obvious mechanisms; in particular Z is a rocker shared between two nozzles.

| Index | ID    | Kind   | Scale     | Mechanism                                         |
|-------|-------|--------|-----------|---------------------------------------------------|
| 0     | peel  | linear | 1.0       | Shared peel-tape take-up motor (all feeders)      |
| 1     | z     | rocker | 1.0       | Z plunge; rocker shared between nozzle 1 and 2    |
| 2     | x     | linear | 32.8084   | Gantry X                                          |
| 3     | a1    | rotary | 4.4444    | Head 1 rotation (theta)                           |
| 4     | y     | linear | 32.8084   | Gantry Y                                          |
| 5     | a2    | rotary | 4.4444    | Head 2 rotation (theta)                           |

## Position encoding

A position on the wire is a signed int32 (little-endian) computed as:

```
wire = round(mm * scale[i]) * 1000
```

Inverse (decode):

```
mm = (wire / 1000) / scale[i]
```

The `*1000` factor is the controller's internal micron representation. The intermediate truncation to integer microns quantises rotary positions to about 0.225° (the rotary scale is 4.4444 steps per degree, so 1 micron = 1/4444.4 ° ≈ 0.000225°; but the round-before-multiply step quantises to 1 µm in scaled units).

## The rocker Z

Axis 1 is **one Z drive** that powers both nozzles through a mechanical seesaw rocker (~17 mm crank arm, 1600 steps per revolution). Lowering nozzle 2 raises nozzle 1; only one nozzle is down at a time.

**The protocol does not expose this coupling.** From the wire's perspective, axis 1 is a single rotary actuator. Translating its angle into "nozzle N is at height Z" is mechanism-side math:

```
z_nozzle1 =  17 mm * sin(angle)
z_nozzle2 = -17 mm * sin(angle)
```

Reimplementations targeting OpenPnP can model this with two `CamAxis` transforms over a single underlying axis — one clockwise, one counter-clockwise — both with a cam radius of 17 mm.

## Coordinate frames

The protocol uses raw machine coordinates. Positions are in the controller's reference frame (X, Y from gantry home; rotation referenced to the controller's zero, not to part orientation).

There is **no** part / placement / fiducial transform in the protocol. All vision corrections happen on the host side and are applied as adjusted move targets in machine coordinates.

## Axis state byte

For each axis, the status poll response carries one byte (offsets 36..41). The safe rule is:

- `0` = idle (axis is at its commanded position and not moving)
- `non-zero` = busy (moving, homing, or in error)

The four nonzero codes (1..4) are observed in the wild but treating them as opaque "busy" is the robust strategy.

## Limits

X and Y have positive AND negative limit switches; Z has one home / limit. The limit bits live in the status-poll input bytes; see [`06-io-map.md`](06-io-map.md). Limit hits do not auto-stop motion on the controller — the host must detect the bit and issue `stop` (opcode 0x09).
