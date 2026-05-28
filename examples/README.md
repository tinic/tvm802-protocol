# Examples

Each file is an annotated hex dump of one request/response round-trip on the wire. Bytes are shown in transmission order; offsets are zero-based from the start of the frame.

Format:

```
== description ==
direction (host → ctrl  or  ctrl → host)
hex bytes
explanation per offset
```

All examples are real captures from a TVM802B, reproducible against a stock controller.

| File                       | Demonstrates                                  |
|----------------------------|-----------------------------------------------|
| `status-poll.hex`          | The 0x00 polling round-trip; full decode.     |
| `read-param.hex`           | Reading the down-camera X scale (key 34).     |
| `write-param.hex`          | Writing + committing a param.                 |
| `move.hex`                 | Immediate move on X+Y.                        |
| `set-output.hex`           | Single-bit output toggle (light ring on).     |
| `feeder-advance.hex`       | One pick FSM run on the LEFT bank.            |
