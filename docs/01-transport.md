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

## Endpoint discovery

The default endpoint (`192.168.0.8:701`) is hardcoded in the stock host and re-advertised by the controller itself as parameters `key 2` (IP, packed little-endian IPv4) and `key 3` (port). A re-configured controller can still be discovered by:

- Reading the host process's outbound TCP table to find an established connection to port 701, or
- Probing the broadcast / link-local subnet for a port-701 listener.

This spec does not mandate either; default-endpoint hardcoding is acceptable for most installs.
