# TVM802 Protocol

A complete, reimplementation-grade specification of the **Qihe TVM802** pick-and-place machine's motion-controller TCP protocol.

Covers the TVM802A / TVM802B / TVM802B+ family (all known variants share this protocol).

## Layout

```
spec/
  protocol.yaml         Machine-readable single source of truth.
docs/
  01-transport.md       TCP framing, endpoint, ACK rule, drain.
  02-opcodes.md         Opcode summary table.
  03-status-poll.md     The big op-0 response decoded.
  04-axes.md            6-axis model + rocker-Z.
  05-motion.md          Queue + poll completion model.
  06-io-map.md          Digital outputs + inputs.
  07-parameters.md      Param keys and read/write.
  08-feeders.md         Prick + peel + drag advance.
  09-homing.md          Homing sequence.
  10-camera-mux.md      Up/down camera switching.
  11-jog.md             Continuous-motion (button-held) jog model.
examples/
  *.hex                 Annotated wire captures.
```

The YAML is the source of truth; the Markdown docs explain *why*.

## Status & confidence

Field offsets and opcode semantics in `spec/protocol.yaml` are verified against live wire traffic on a TVM802B (firmware reporting key 3 = 701, key 34 = 268, key 35 = 253). Per-field confidence is noted in the YAML where it isn't `verified`.

If you reimplement against this and find a discrepancy, please open an issue with a wire capture.

## Transport one-liner

TCP. Controller is the server on `192.168.0.8:701` (factory default). Little-endian everywhere. No checksum, no length prefix — frame size is fixed per opcode, determined by `byte[0]`. ACK is a response whose `byte[0]` echoes the request.

## License

MIT. See [`LICENSE`](LICENSE).

This is an independent specification derived from observed wire traffic and probe testing of a TVM802B. It contains no vendor source code or binaries.
