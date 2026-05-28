# Host design notes

The protocol gives you bytes on the wire. A reimplementation also needs operational patterns — the things you only learn by running a TVM802 long enough to hit the edge cases. This page collects them.

## Missing-part reject: vacuum check vs vision size-check

Two mechanisms can detect that a pick attempt failed. They have very different operational characteristics.

### Vacuum check (the graceful one)

Read the head's vacuum analog channel from `status_poll` (bytes 42-43 for head 1, 44-45 for head 2). Compare against a baseline:

```
delta = baseline_raw - vacuum_raw
if delta < part_threshold_raw:
    skip placement, continue with next part
```

Behaviour: the host marks the slot as "done" and moves on. No stop, no dialog, no operator intervention required. Just a gap on the board where the part would have been.

This is the recommended missing-part handling path for any reimplementation. It uses the definitive presence signal (vacuum pressure), it's fast, and it doesn't halt the line.

### Vision size-check (the halting one)

Capture an up-vision frame after pick, run a component detector, compare the detected bounding-box against the configured part W/H within a per-part tolerance (typically 10 %).

Behaviour on failure: retry the pick up to 3 times (~6 seconds each), then **hard-stop** the line with a "VisionSizeError" dialog. Operator must clear the dialog and intervene.

This is the wrong tool for missing-part reject — it costs ~18 seconds of retry before surfacing the error, and the error is operator-disruptive. Use it (if at all) for *wrong* parts (part present but visibly the wrong component), not for *missing* parts.

### Master gate

The host's "use missing check" master gate is **bit 0 of parameter key 23** (the feature-flag byte). Per-part check method lives in the placement job data (not in the controller param store) — typically a UI dropdown with options "none / pressure / vision / both".

For a clean reimplementation: master gate **on**, per-part **pressure**, configure vacuum thresholds in keys 54 / 55 (one per head channel, per A/B preset).

## Vacuum threshold calibration

The vacuum analog is a raw 16-bit ADC reading. Calibration is per-machine:

1. With the head at rest and no part loaded: read `vacuum_*_raw` from `status_poll` over several seconds and take the mean → **baseline**.
2. Pick a typical part: read `vacuum_*_raw` while held → **loaded**.
3. Set the threshold to `0.4 × (baseline - loaded)` as a starting point. This gives a comfortable margin for healthy picks while still failing dropped picks.

Re-calibrate after any pump / hose maintenance — the baseline drifts with seal condition.

## Size-gate orientation

The host's vision size-check sorts BOTH detected and expected dimensions to `(max, min)` before comparing. This means **W / H labelling is orientation-invariant**: a 4×2 part reported as `(W=2, H=4)` passes the gate identically to one reported as `(W=4, H=2)`. Vision detectors do not need to normalise long-axis-as-W.

If you implement your own vision pipeline, report whatever pose comes out of your detector. The size-gate doesn't care which axis you call "W".

## Settle-aware capture

The down-vision camera has no software settle delay built into its frame grab. If a vision step samples while the mount is in motion, the deinterlaced frame shows **comb artefacts** (consecutive interlace fields captured different scenes). The detector reads off-target features or fails entirely.

For closed-loop vision (correction loops, jog-during-vision):

1. Stop the mount axis (opcode `0x09`).
2. Poll `status_poll` until `axis_state[axis] == 0` for two consecutive polls.
3. Wait one additional frame period (~33 ms for NTSC, ~40 ms for PAL) to flush the in-flight USB buffer.
4. Then grab.

For static captures (mount already settled before the vision step), step 3 alone is enough.

## Camera-switch settle

The CD4052 analog mux relay settles in microseconds, but the USB grabber's frame buffer is one or two frames behind. After flipping `output bit 0x8000`:

1. Wait at least 50 ms.
2. Discard the next 1-2 frames.
3. Grab the third frame.

A naive "switch and capture immediately" path produces a frame from the previous camera roughly half the time.

## Z-rocker safety

Axis 1 is a mechanical rocker shared between both nozzles. Lowering nozzle 2 raises nozzle 1, and vice versa. There is no "both up" safe state via a single axis 1 position — there is one "centre" position where both are at maximum lift, and that's where you want the head during XY moves.

Plan placements so that:

1. Before any XY move, axis 1 is at the rocker centre (`axis_position[1]` close to 0 or a calibrated zero, depending on home convention).
2. The "park" position has both nozzles up.
3. Z plunges happen ONLY after XY is at the target. Never plunge while moving.

The host enforces this by serialising motion in its FSM, not by any protocol-level guarantee. A reimplementation must do the same.

## Connection robustness

The motion controller's TCP server has narrow timeouts (100 ms send/recv on the stock host) and accepts a maximum of 2 retries per request. On poor network conditions (e.g. PoE / industrial switch with PLC traffic priority), a reimplementation may want to:

- Increase the timeouts to 250-500 ms on commodity Ethernet.
- Implement a connection-monitor task that reconnects automatically on socket failure, re-issues opcode `0x05`, and resumes the polling loop.
- NOT auto-retry motion commands across a reconnect — the controller's FIFO is empty after a reconnect and any in-flight queued motion was lost. Re-plan the placement step.

The 100 ms stock timeout is fine for a directly-connected wired link but tight for anything else.

## Vacuum pump state

The vacuum pump is on output bits `0x008` + `0x400` (the host writes `0x408` to enable). The pump runs continuously while bit `0x408` is set; both head vacuum solenoids switch the actual head suction on / off independently (`0x004` head 1, `0x2000` head 2).

For low-throughput sessions, leave the pump on at job start and toggle the solenoids per pick. For maintenance sessions or pump-cycle stress reduction, toggle the pump on demand. The protocol does not constrain either pattern.

## Job-start sequence

A "start a job" sequence the host typically emits:

```
status_poll                                    # confirm machine is idle
output_bit_on_immediate  bit = 0x0020          # run indicator on
output_bit_on_immediate  bit = 0x0408          # pump on
set_motion_profile_immediate ...               # job-profile speeds
# Then per-part:
move_buffered ... (to feeder)                  # head to pick slot
feed sequence ...                              # see docs/08-feeders.md
move_buffered ... (to up-camera)               # vision step
move_buffered ... (to PCB target)              # place
output_bit_off_queued bit = 0x0004 (or 0x2000) # release vacuum (place)
output_bit_on_queued  bit = 0x0002 (or 0x1000) # blow-off
# Loop until parts exhausted, then:
output_bit_off_immediate bit = 0x0408          # pump off
output_bit_off_immediate bit = 0x0020          # run indicator off
move_immediate ... (to home)
```

Exact ordering varies (some hosts park the head between every pick; others serialise more aggressively). The protocol does not prescribe a job-loop shape — it gives you the building blocks.

## Logging / telemetry

The protocol has no built-in logging stream. For diagnostics:

- Log every `status_poll` response that contains a non-zero `event_code` (byte 46).
- Log axis-state transitions (per-axis `axis_state` byte) — useful for "where did the move actually fail".
- Log queue depth (`status_poll.queue_free`) at job-loop ticks — sudden drops to 0 hint at a stalled FIFO.
- Log vacuum readings (`vacuum_1_raw`, `vacuum_2_raw`) at each pick step — invaluable when tuning vacuum thresholds.

A round-trip pcap of the TCP traffic is also extremely useful when debugging; the protocol is simple enough that a 10-second capture covers most failure modes.
