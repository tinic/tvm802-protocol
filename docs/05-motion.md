# Motion-completion model

The controller does NOT send a per-move ACK. There is no "move done" frame. Instead:

1. The host **polls** `status_poll` (opcode 0x00) every UI tick.
2. The host **queues** subsequent moves; the controller drains the queue while it has free FIFO slots.
3. "Move done" is **inferred** from the polled state.

This is the central flow-control idiom of the protocol. Get it right and everything else falls into place.

## Move-done predicate

For a target on axis `i`:

```
done(i) = (axis_state[i] == 0) AND (axis_position[i] == target[i])
```

Both conditions must hold. `axis_state == 0` alone is not enough — an axis can read idle for one poll cycle while the controller is between sub-moves of a coordinated motion.

For a multi-axis move (axis_mask covering multiple bits), the move is done when the predicate holds for every axis in the mask.

## The queue + the FIFO

The controller has a small command FIFO (~7 slots). Queued opcodes (`0x0C move_buffered`, `0x0D feed_axis0`, `0x0E set_outputs_queued`, etc.) are appended to a **host-side list**, then flushed into the controller's FIFO opportunistically:

```
on each status_poll response:
    free = response.queue_free
    while free > 0 and host_queue not empty:
        send host_queue.pop_front()
        free -= 1
```

This means the controller is always 1..7 commands ahead of the host. Long sequences of small queued moves can be planned without waiting for each one to complete; the host just keeps the queue topped up.

**Do not flush more than `queue_free` commands per poll.** The controller has no flow-control feedback beyond the next `queue_free` reading.

## Why not just busy-wait?

You can't. A blocking "wait until move done" implementation would have to either:

- Spin on `status_poll` between moves — wasteful and serialises the FIFO.
- Block until some imagined per-move ACK arrives — which doesn't exist.

The right shape is a state machine:

```
state := IDLE
on tick:
    response = status_poll()
    if state == MOVING and done(current_target):
        state = IDLE
        advance_program()
    drain_host_queue(response.queue_free)
```

This is what the stock host does. A reimplementation should do the same.

## Immediate vs queued

Use **immediate** opcodes for:

- UI jog (start / stop on button press / release)
- Stop / abort
- Diagnostic reads (`status_poll`, `read_param`)
- One-shot output toggles tied to a UI button

Use **queued** opcodes for:

- Everything inside a placement cycle (move, dwell, feeder advance, output bit toggles that sequence with motion)
- Anything that should pipeline with the controller staying ahead

A queued opcode does NOT block the host's next `send()` — there is no synchronous handshake. The host just appends and continues.

## Stopping mid-move

To stop a queued sequence:

1. `0x09 stop` with axis_mask covering all moving axes (or `0x3F` for all).
2. `0x0B program_control` with command `0` to clear the queue.
3. Discard the host-side queue.

Until you do step 2, the controller will resume draining the FIFO when stop is released. This is a frequent source of "phantom motion" bugs in early reimplementations.

## Stale-position quirk

After a queued move completes, the position reported by `status_poll` is the actual encoder position, which may differ from the integer target by 1 raw unit due to round-trip rounding. The stock host treats `|axis_position - target| <= 1` as done. A reimplementation should do the same.
