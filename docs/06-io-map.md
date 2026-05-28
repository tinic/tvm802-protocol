# Digital I/O map

The controller has a 32-bit digital-output bitmap, written with opcodes `0x04` / `0x0E` (full word) or `0x14`-`0x17` (per-bit set / clear). It also has digital and analog inputs that are read **only via the status poll** — there is no separate input-read opcode.

## Outputs

| Mask     | ID                | Notes                                                                 |
|----------|-------------------|-----------------------------------------------------------------------|
| `0x0001` | `cam_light_pair`  | Camera-light pairing enable. Required with `lamp_enable` to light the ring. |
| `0x0002` | `head1_blow`      | Head 1 blow-off / release (positive air, ejects part).                |
| `0x0004` | `head1_vacuum`    | Head 1 vacuum solenoid.                                               |
| `0x0008` | `vacuum_pump_a`   | Vacuum pump enable (paired with `0x400`).                             |
| `0x0020` | `run`             | Run / start-cycle indicator (operator panel light).                   |
| `0x0040` | `suspend`         | Suspend / pause indicator (operator panel light).                     |
| `0x0080` | `jog_mode_a`      | Jog-speed selector (with `0x100`).                                    |
| `0x0100` | `jog_mode_b`      | Jog-speed selector (with `0x080`). Together `0x180` = high-speed.     |
| `0x0200` | `lamp_enable`     | Light-ring power enable.                                              |
| `0x0400` | `vacuum_pump_b`   | Vacuum pump enable (with `0x008`). Pump-on word = `0x408`.            |
| `0x0800` | `buzzer`          | Audible alarm.                                                        |
| `0x1000` | `head2_blow`      | Head 2 blow-off / release.                                            |
| `0x2000` | `head2_vacuum`    | Head 2 vacuum solenoid.                                               |
| `0x4000` | `prick_pin`       | Prick / pull-pin actuator on the moving head; drags carrier tape.     |
| `0x8000` | `cam_mux`         | Camera up/down mux relay (CD4052).                                    |

### Common composite values

| Value    | Meaning                                |
|----------|----------------------------------------|
| `0x0201` | Light ring ON (lamp_enable + pair).    |
| `0x0408` | Vacuum pump ON.                        |
| `0x8200` | Camera UP selected (mux + lamp).       |
| `0x0180` | Jog: high-speed mode.                  |

### Bits unassigned

Masks not listed (e.g. `0x10`) are unused in production. Writing them is harmless but reads back as written; do not rely on them.

## Inputs (read from `status_poll`)

Input bits live in the four bytes at status-response offsets 8, 9, 10, 11. Most inputs are **active LOW**.

### Limit switches

| Bit         | ID            | Active                       |
|-------------|---------------|------------------------------|
| byte 8 bit 2| `x_limit_pos` | LOW (limit hit pulls low)    |
| byte 8 bit 3| `x_limit_neg` | LOW                          |
| byte 8 bit 4| `y_limit_pos` | LOW                          |
| byte 8 bit 5| `y_limit_neg` | LOW                          |
| byte 9 bit 1| `z_home_limit`| LOW                          |
| byte 11 bit 6| `prick_home` | LOW                          |

### Operator panel buttons

The front-panel buttons report as inputs (polling). The host can either use them directly or pass them through; they are reported regardless.

| Bit          | Button                          |
|--------------|---------------------------------|
| byte 10 bit 6| Y-                              |
| byte 10 bit 7| X+                              |
| byte 11 bit 0| Step                            |
| byte 11 bit 1| Stop / Suspend                  |
| byte 11 bit 2| Run / Start                     |
| byte 11 bit 3| X-                              |
| byte 11 bit 4| Jog-speed select                |
| byte 11 bit 5| Y+                              |

## Analog inputs (vacuum)

Vacuum-pressure sensors for the two heads are read from the status response at bytes 42-45 (`vacuum_1_raw`, `vacuum_2_raw`, u16 LE).

These are raw 16-bit ADC values. The "pickup ok" decision is host-side:

```
delta = baseline_pressure - reading
if delta < part_threshold:
    raise pickup_failed
```

The baseline and per-part threshold are not in the controller. Calibrate at idle (head up, no part) to find baseline, then characterise loaded readings per part type.

## No E-stop bit

The protocol does **not** expose an emergency-stop input. E-stop is wired into the controller's hardware and halts motion at a level below the network layer. A reimplementation cannot detect E-stop from the wire alone — the host learns of an E-stop only by observing that motion didn't happen and the system entered an error state.
