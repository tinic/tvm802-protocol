# Transport

The controller is a **TCP server**. The host is a TCP client.

| Property              | Value                                          |
|-----------------------|------------------------------------------------|
| Protocol              | TCP                                            |
| Default endpoint      | `192.168.0.8:701`                              |
| Byte order            | Little-endian (all multi-byte fields)          |
| Framing               | Fixed size per opcode (no length prefix)       |
| Checksum              | None                                           |
| Concurrent clients    | Yes (verified — second reader does not disturb host) |
| `TCP_NODELAY`         | On                                             |
| Send timeout          | 100 ms                                         |
| Receive timeout       | 100 ms                                         |

## Frame layout

Every request begins with a 4-byte header:

| Offset | Field    | Type | Meaning                                                    |
|--------|----------|------|------------------------------------------------------------|
| 0      | opcode   | u8   | Selects command; determines total frame size               |
| 1      | sequence | u8   | Sequence counter (queued commands); 0 for immediate        |
| 2      | aux      | u8   | Axis-mask or sub-argument (opcode-specific)                |
| 3      | pad      | u8   | 0                                                          |

Total frame size is **fixed per opcode** — there is no length prefix. A reimplementation must look up the size from `byte[0]` and read exactly that many bytes.

## ACK rule

A response whose `byte[0]` equals the request's `byte[0]` is the ACK for that request. There is no per-command status byte. The host:

1. **Drains** any pending bytes on the socket.
2. Sends the request.
3. Reads frames; discards any whose `byte[0]` does not match the request opcode.
4. The first matching frame is the ACK.

A reimplementation MUST drain on every request; otherwise the first read after connect will pick up stale/banner bytes and misalign.

## Multi-client behaviour

The controller accepts at least two concurrent TCP clients on port 701. A passive monitor / parameter reader can connect alongside the live host with no observable disturbance, provided the second client only issues read-style opcodes (0x00 status_poll, 0x01 read_param). Writing parameters or commanding motion from a second client while the host is running is **not** safe and is not covered by this spec.

## Sequence counter

`byte[1]` of every request is a sequence counter. The host:

- Writes `0` for every **immediate** command.
- Maintains a running `u8` counter for **queued** commands; the counter increments on each frame popped off the host-side outgoing list. The counter wraps at `0xFF`.

The controller does not appear to validate the counter. A reimplementation that always sends `0` will still work. The canonical behaviour (increment in queued frames) is documented for completeness.

## Connection lifecycle

A fresh connection looks like this:

1. Open TCP socket to the controller endpoint (`192.168.0.8:701` default).
2. Set `TCP_NODELAY`; configure 100 ms send and receive timeouts.
3. Send **opcode `0x05`** (`controller_init`) ONCE. This is the servo-enable.
4. Read any startup parameters (typically keys `2` / `3` for IP / port sanity, plus camera scales and home positions for the host's own caches).
5. Begin the `status_poll` (`0x00`) cadence at ~50 ms.

After this, the host issues motion / I/O commands as user actions dictate. Opcode `0x05` is **not** re-sent before every motion — it is a one-shot per connection.

On **reconnect** after a socket failure:

1. Re-open the TCP socket.
2. Re-issue opcode `0x05` to re-arm the servos.
3. Resume the `status_poll` loop. The controller's command FIFO is discarded across a disconnect; the host's queue must be reset on the host side too.

## Retry and timeout policy

Per-request:

- The host's send-and-ack function attempts the request **at most 2 times** (one initial + one retry on misalignment).
- On total failure, the function returns false; the caller surfaces an error and aborts the in-flight operation.
- On socket exception (connection lost), the host hard-fails the request — there is no automatic backoff or queued retry on the wire.

For long-running motion, the host pre-empts timeouts by polling `status_poll` at fixed intervals; a missed poll causes a re-connect attempt on the next tick rather than a tight retry loop.

## Endpoint discovery

The default endpoint (`192.168.0.8:701`) is hardcoded in the stock host and re-advertised by the controller itself as parameters `key 2` (IP, packed little-endian IPv4) and `key 3` (port). A re-configured controller can still be discovered by:

- Reading the host process's outbound TCP table to find an established connection to port 701, or
- Probing the broadcast / link-local subnet for a port-701 listener.

This spec does not mandate either; default-endpoint hardcoding is acceptable for most installs.
