# Feeders

The TVM802 does **not** have per-feeder motors. The feeding mechanism is:

1. A shared **peel-tape take-up motor** (axis 0) that pulls cover tape off ALL active feeders simultaneously through a friction shaft.
2. A **prick / pull-pin actuator** on the moving head (output bit `0x4000`).
3. The **head itself**, used to drag carrier tape one pitch per feed.

A "feed one part" is a *head motion*, not a feeder motion. The head positions over the feeder, pushes the prick pin down into a sprocket hole on the carrier tape, drives X or Y by one tape pitch (typically -1500 or -3000 raw steps depending on part size), then retracts the pin while axis 0 takes up the slack on the peel tape.

## Layout

Feeders sit on **two banks** of the machine:

- **LEFT bank**: tape feeders, drag axis = **X**
- **BACK bank**: tape feeders, drag axis = **Y**

Plus a small number of front tray stacks (no advance — the head just picks the next exposed part).

The exact count and positions are machine-specific and live in the host's `Param.dat` file alongside parameter overrides for the vision system. The protocol itself has no notion of "feeder N" — the host issues a head-drag move to a specific machine coordinate and a peel advance, and that *is* the feed.

## Feed sequence

For one part advance:

```
0x12 teach_position_queued  axis_mask=0x01, targets[0]=0   # zero axis 0
0x16 output_bit_on_queued   bit=0x4000                      # extend prick pin
0x13 pick_sync                                              # sync marker
0x11 dwell                  ms=N                            # settle (N small)
0x0C move_buffered          axis_mask=0x04 or 0x10          # head drags one pitch
                            targets = current ± pitch
0x0D feed_axis0             axis_mask=0x01, targets[0]=-S   # peel take-up
0x17 output_bit_off_queued  bit=0x4000                      # retract prick pin
```

Pitch values commonly seen:

| Part            | Pitch (raw, signed) |
|-----------------|---------------------|
| 0402 / 0603     | -1500               |
| 0805 / 1206     | -3000               |
| Larger          | -3000 to -6000      |

The sign matters: feeds always move in the negative direction along the drag axis.

## Why this scheme?

Cost. Per-feeder motors triple the machine BOM. The shared peel + head-drag scheme keeps everything mechanical on the feeder slot itself (just a sprocket and a tensioner), at the price of a head trip back to the feeder before every pick. This is the central design tradeoff of the TVM802 family.

## Tray feeders vs tape feeders

Front-tray stack feeders (Table E in [`07-parameters.md`](07-parameters.md), parameter keys 420-449) are **fundamentally different** from tape feeders:

| Aspect             | Tape feeders                       | Tray feeders                                |
|--------------------|------------------------------------|---------------------------------------------|
| Advance mechanism  | Prick + head drag + peel           | Purely positional — no advance opcode       |
| Per-pick motion    | Head moves to pitch + drag         | Head moves to *next stack position*         |
| Prick pin          | Used (output bit `0x4000`)         | Not used; prick stays retracted             |
| Peel axis          | Used (axis 0)                      | Not used                                    |
| Per-feeder state   | Tape position (advances over time) | Stack-index (decrements as parts taken)     |
| Stock count source | Implicit from tape pitch + count   | Per-tray count param (in Table E packing)   |

For a tray pick, the host:

```
0x06 set_motion_profile_immediate     axis = X / Y, profile = pick-approach
0x08 move_immediate                   target = (tray_pos_x, tray_pos_y) from stack-index
0x06 set_motion_profile_immediate     axis = Z, profile = plunge
0x08 move_immediate                   target = (current_x, current_y, nozzle_z_front_stack)
0x14 output_bit_on_immediate          bit = head_N_vacuum
0x11 dwell                            ms = settle
0x08 move_immediate                   target = (current_x, current_y, safe_z)
```

No opcode `0x0D feed_axis0`, no prick-pin toggle, no head drag move. The "advance" is just: next time we want a part from this tray, use the next stack position.

A reimplementation needs separate feeder classes (or a class-with-mode) for tape vs tray; the protocol-level sequences share opcodes but differ in shape.

## Drop-bin / reject

There is no drop-bin sequence in the protocol. The stock host implements "reject" as:

1. Drive head to the `discard` position (param keys 12 / 13).
2. Pulse blow-off (output bit `0x0002` or `0x1000` depending on head).
3. Return to flow.

This is purely host-side composition of motion + output toggles. The protocol provides no dedicated reject opcode.

## Pickup verification

After a feed-and-pick, the host typically issues a short dwell and reads the vacuum analog channel for the corresponding head (`vacuum_1_raw` or `vacuum_2_raw` in the status poll response). The "pickup ok" predicate is:

```
delta = baseline - reading
if delta < threshold: pickup_failed → retry or skip
```

The threshold is part-specific. Most users tune it once per part type by picking and reading.
