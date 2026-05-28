# Opcode summary

All sizes include the 4-byte header. `delivery` is `immediate` (sent to controller now) or `queued` (appended to the FIFO, flushed when `status_poll.queue_free > 0`).

| Code   | Name                          | Delivery   | Req size | Resp size | Purpose                                              |
|--------|-------------------------------|------------|----------|-----------|------------------------------------------------------|
| `0x00` | `status_poll`                 | immediate  | 4        | 47        | Read full machine state. The polling frame.          |
| `0x01` | `read_param`                  | immediate  | 12       | 12        | Read one keyed configuration parameter.              |
| `0x02` | `write_param`                 | immediate  | 12       | 12        | Write one keyed parameter (must be committed).       |
| `0x03` | `commit_params`               | immediate  | 4        | 4         | Latch pending param writes.                          |
| `0x04` | `set_outputs_immediate`       | immediate  | 8        | —         | Replace the digital-output word.                     |
| `0x05` | `controller_init`             | immediate  | 8        | —         | Servo-enable / init after TCP connect.               |
| `0x06` | `set_motion_profile_immediate`| immediate  | 76       | —         | Per-axis accel / decel / velocity tables.            |
| `0x07` | `teach_position_immediate`    | immediate  | 28       | —         | Set controller's current-position belief; not a move.|
| `0x08` | `move_immediate`              | immediate  | 28       | —         | Coordinated, masked move.                            |
| `0x09` | `stop`                        | immediate  | 8        | —         | Stop one or more axes.                               |
| `0x0B` | `program_control`             | immediate  | 4        | —         | Clear / run / pause the program queue.               |
| `0x0C` | `move_buffered`               | queued     | 28       | —         | The standard job move (queued).                      |
| `0x0D` | `feed_axis0`                  | queued     | 28       | —         | Feeder advance: relative step on axis 0.             |
| `0x0E` | `set_outputs_queued`          | queued     | 8        | —         | Replace digital-output word, queued.                 |
| `0x0F` | `set_motion_profile_queued`   | queued     | 76       | —         | Queued motion-profile update.                        |
| `0x11` | `dwell`                       | queued     | 8        | —         | Sleep N ms (queued).                                 |
| `0x12` | `teach_position_queued`       | queued     | 28       | —         | Queued form of `teach_position_immediate`.           |
| `0x13` | `pick_sync`                   | queued     | 4        | —         | Pick-FSM sync marker.                                |
| `0x14` | `output_bit_on_immediate`     | immediate  | 8        | —         | Set one or more output bits ON.                      |
| `0x15` | `output_bit_off_immediate`    | immediate  | 8        | —         | Clear one or more output bits.                       |
| `0x16` | `output_bit_on_queued`        | queued     | 8        | —         | Queued form of `output_bit_on_immediate`.            |
| `0x17` | `output_bit_off_queued`       | queued     | 8        | —         | Queued form of `output_bit_off_immediate`.           |

A dash in the response-size column means the operation generates no payload beyond the opcode echo (which still arrives — read and discard it).

## Reserved opcodes

The controller's opcode enum runs 0..23 (i.e. 0x00..0x17). The 22 codes listed above are the ones the stock host actually emits. Two codes inside the enum are declared but never used:

| Code   | Status                     | Notes                                              |
|--------|----------------------------|----------------------------------------------------|
| `0x0A` | Declared, never emitted    | Reserved slot. Behaviour undefined. **Do not send.** |
| `0x10` | Declared, never emitted    | Reserved slot. Behaviour undefined. **Do not send.** |

Codes from `0x18` upward are outside the declared enum entirely. Sending them is undefined and on some firmwares produces desynchronised responses — **do not send**.

A reimplementation that wants to be exhaustive can probe `0x0A` and `0x10` against an idle controller to characterise their behaviour, but this spec does not document them because the stock host doesn't use them and a future firmware revision may repurpose them.

## Immediate / queued twins

Many commands have both an immediate and a queued opcode (set-outputs, motion-profile, teach-position, output-bit-on, output-bit-off). The payload is identical between the twin pair; only the delivery differs:

- **Immediate** opcodes go directly to the controller's command processor.
- **Queued** opcodes append to a host-side list. They are drip-fed to the controller while `status_poll.queue_free > 0`. The controller's FIFO depth is ~7 slots.

Use immediate for one-shot UI actions (jog, stop, set-output toggles from a button); use queued for everything inside a placement cycle so the controller can pre-fetch the next motion while finishing the current one.
