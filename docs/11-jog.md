# Jog

Jog is not a special opcode. It is a normal coordinated move with a velocity profile tuned for handle-like response, terminated by a `stop` when the operator releases the button.

The protocol has no "jog mode" flag. The "jog speed" mode bits in the output bitmap (`0x80`, `0x100`) are operator-panel indicators only — they light the front-panel speed LED and may pre-bias which motion profile the host loads, but they do not change the controller's behaviour directly.

## Button-down sequence

When a jog button is pressed (X+, X-, Y+, Y-, Z up, Z down, A1 rotate, A2 rotate, nozzle 1 down/up, nozzle 2 down/up):

```
0x06 set_motion_profile_immediate     axis = <jog axis>, profile = jog accel/decel/velocity
0x08 move_immediate                   axis_mask = <jog axis bit>,
                                      target = <large value beyond reachable travel>
```

The "large target" trick is how the host gets continuous motion from a coordinated-move opcode. The axis will move at the profile velocity until either:

- The button is released (button-up sequence below).
- A limit switch is hit (controller event code 2, host raises an e-stop).
- The (unreachable) target is somehow reached (never happens in practice).

Velocity in the motion profile is **multiplied by 100%** for jog (the speed-percent parameter to `set_motion_profile_immediate` is hard-coded to 100). The jog velocity itself is the profile's velocity field — pre-set in the host's UI / settings, not on the wire.

## Button-up sequence

When the button is released:

```
0x09 stop axis_mask = <jog axis bit>
```

That's it. One frame, immediate. The controller halts the axis using the profile's deceleration; the host's status-poll will show `axis_state == 0` once the deceleration completes.

## Multi-axis jog

If the operator presses two direction buttons simultaneously (e.g. X+ and Y+ for a diagonal), the host issues two button-down sequences. Each axis is profile-loaded and started independently. On release, two separate `stop` frames are sent — one per axis.

There is no "coordinated diagonal jog" via a multi-axis target; the host treats each axis as an independent jog.

## Z (nozzle) jog

Z is the rocker axis shared between the two nozzles. Pressing "Nozzle 1 Down" lowers nozzle 1 (and raises nozzle 2 by the rocker geometry); pressing "Nozzle 2 Down" does the opposite.

Protocol-wise these are normal `0x06 / 0x08` sequences on axis 1; the rocker math is handled by the mechanism, not by special opcodes. See [`04-axes.md`](04-axes.md) for the rocker model.

## Rotation jog

The A1 / A2 buttons drive axes 3 / 5 respectively. Profile velocity is in steps/sec; with the rotary scale of 4.4444, a velocity of 10000 steps/sec works out to about 2250 °/sec — much too fast for handheld jog. The host's jog profile typically has the velocity dialled down by an order of magnitude or more.

## Reimplementation notes

- Treat jog as a stateful UI mode in your reimplementation: on button-down, allocate a "currently jogging axis" slot; on button-up or fault, release it.
- Always send `stop` on button-up. If the operator clicks rapidly the host may end up with a `stop` for an axis that is already idle — that is harmless.
- If the connection drops mid-jog, the controller will keep moving until it hits a limit. **Recovery requires a re-connect, opcode `0x05` re-init, and an explicit `0x09 stop`** before the system is safe to issue any further commands.
- Polling `status_poll` while jogging gives you live position for the UI position display; the cadence is the same ~50 ms as normal operation.
