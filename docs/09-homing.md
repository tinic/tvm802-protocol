# Homing

The controller does not auto-home on power-up. The host runs an explicit per-axis sequence using `move_immediate`, the limit-switch input bits, `stop`, and `teach_position`.

## Per-axis homing sequence

For one linear axis (X or Y):

```
1. Set fast motion profile         (opcode 0x06, accel/vel for approach)
2. Move toward limit               (opcode 0x08 move_immediate, large -ve target)
3. Loop:
       poll status (opcode 0x00)
       if limit bit asserted:
           break
4. Stop axis                       (opcode 0x09 stop, mask = axis bit)
5. Set slow motion profile         (opcode 0x06, slow back-off speeds)
6. Move OFF the limit              (small +ve target)
7. Move back ONTO the limit slowly (small -ve target, again to the switch)
8. Teach zero                      (opcode 0x07 teach_position, target=0)
```

The two-pass approach (fast then slow) is necessary because the fast approach overshoots the switch by several encoder counts; the slow re-touch gives a repeatable zero.

## Axis order

The stock host homes in this order:

1. **Z** first (if Z-equipped; gated by parameter `45 = feature_flag_v3_09`).
2. **X**.
3. **Y**.

Z first is a safety measure — it ensures both nozzles are at a known height before any X/Y motion happens. Some early firmware (≤ 3.09) lacks the Z home limit and skips this step.

## Z homing

Z homing is more involved because of the rocker:

1. Read param `45` — if false, skip Z home.
2. Drive axis 1 to the side that lifts both nozzles (towards the rocker centre).
3. Watch `z_home_limit` (byte 9 bit 1 of status response).
4. On assertion: `stop`, back off, re-approach slowly.
5. Apply the Z home offset (`param 56`) by issuing `teach_position` with that offset as the target — this becomes axis 1's zero.

After Z home, both nozzles are at the controller-calibrated "tip just above PCB plane" height.

## Failure modes

- **Limit not reached** → soft timeout. The host aborts after N seconds of motion and surfaces an error.
- **Limit asserted before motion** → either the switch is broken or the axis is already on the limit. The host backs off slightly and re-attempts.
- **Teach with wrong polarity** → axis zero is at the wrong end. This is a recoverable mistake on a sandboxed machine but can crash the head on a real one. Always validate homing on a fresh reimplementation by jogging carefully BEFORE issuing the first `teach_position`.

## After homing

The host:

1. Reads home position params (keys 4 / 5) and stores them as the "park" location.
2. Sets the motion profile back to job defaults.
3. Optionally issues a `move_immediate` to park.

There is no protocol-level concept of "homing done" — the host advances its own FSM after the last teach completes successfully.
