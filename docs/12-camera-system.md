# Camera system

The TVM802 has two analog cameras (top-down board-fiducial + bottom-up component-vision) that share **one** USB capture device. Driving them from a host involves three distinct interfaces:

1. **Controller TCP protocol** (the rest of this spec) — output bits select which camera the mux relays, and turn the LED ring on.
2. **USB video stream** — a single composite YUV stream from the mux-output, delivered via the USB grabber as standard UVC-like frames.
3. **USB control transfers** — vendor-specific requests that let the host write to the analog video decoder's I²C registers (gain, brightness, contrast, hue, saturation).

A reimplementation that wants more than the stock host has must drive all three.

## Hardware chain

```
+----------+    +-----------+    analog CVBS    +----------+    digital YUV    +----------+    USB    +------+
|  Camera  |--->| CD4052    |------------------>|  SAA7113 |------------------>|  STK1160 |---------->| Host |
|  (up)    |    | analog    |                   |  decoder |    8-bit UYVY     |  bridge  |  control  |      |
+----------+    | mux       |                                                                                |
+----------+    |           |                                                                                |
|  Camera  |--->|           |                                                                                |
|  (down)  |    |           |                                                                                |
+----------+    +-----------+                                                                                |
                     ^                                                                                       |
                     | mux select                                                                            |
                +-------------+    TCP (this spec)                                                           |
                | Motion ctrl |<------------------------------------------------------------------------------+
                +-------------+
```

Each arrow has its own protocol:

| Hop                       | Protocol            | Bandwidth        |
|---------------------------|---------------------|------------------|
| Camera → CD4052           | Analog CVBS         | NTSC/PAL line rate |
| CD4052 → SAA7113          | Analog CVBS         | Same             |
| SAA7113 → STK1160         | 8-bit ITU-R BT.656  | ~27 MHz pixel clock |
| STK1160 → Host (video)    | USB 2.0 isochronous | UYVY 640×480 @ 25 fps (PAL) or 30 fps (NTSC) |
| STK1160 → Host (control)  | USB 2.0 control     | Bulk, single-shot vendor requests |
| Host → Motion controller  | TCP (this spec)     | < 1 KB/s         |

The chain is **analog** until the SAA7113's ADC. Anything before that point (lighting, mux quality, cable length, camera SNR) is the floor on image quality; the digital pipeline can only preserve, not improve, what the analog stage delivered.

## Identifying the grabber

| Property      | Value                                            |
|---------------|--------------------------------------------------|
| USB VID / PID | `0x05E1` / `0x0408`                              |
| Product name  | `USB2.0 Grabber`                                 |
| Chip          | Syntek STK1150 (or STK1160; identical USB ID)    |
| UVC compliant | Partial — exposes video stream interfaces but not standard `IAMCameraControl` |
| Pixel formats | **YUY2 / UYVY 640×480 only**; RGB enumerates but `openStream` fails |
| Output depth  | 8 bits per channel (chip is 10-bit internal; the higher bits are not exposed over USB) |
| Frame rate    | 25 fps (PAL chain) or 30 fps (NTSC chain), interlaced source de-interlaced by the SAA7113 |

The Windows driver bundled with the device targets the M811 CMOS-sensor family (not the SAA7113 analog path); on this machine it binds successfully on USB-ID match but its proc-amp controls (Brightness, Gain, etc.) act on a CMOS pipeline that is not in the signal path. **Standard `IAMVideoProcAmp` controls are software-faked.** A reimplementation that wants real hardware tuning must bypass the bundled driver and talk to the chip directly via WinUSB (Windows) or `libusb` (Linux).

The Linux kernel `stk1160` driver (`drivers/media/usb/stk1160/`) is the practical documentation for the chip's USB interface — same VID/PID, same register map.

## Camera selection (controller mux)

See also [`10-camera-mux.md`](10-camera-mux.md).

The mux is a CD4052 analog relay driven by **controller output bit `0x8000`**:

| Bit `0x8000` | Camera selected | Composite output word used by host |
|--------------|-----------------|--------------------------------------|
| Clear        | Down (board)    | `0x0201` (light + pair, mux clear)   |
| Set          | Up (parts)      | `0x8200` (mux + lamp)                |

LED ring light enable lives in bits `0x0200` (lamp) and `0x0001` (camera-light pairing). The host writes both together as `0x0201` to light the ring with the down camera and `0x8201` (mux + lamp + pair) for the up camera.

After the controller flips the mux relay, the **USB stream content changes** without any reconfiguration of the USB device — the camera switch is invisible to USB. The chain is "same device, different upstream input." Settle time after the relay flip is hardware-limited; allow ~50 ms before sampling pixels to clear the analog transient and any frame that was in flight pre-flip.

## STK1160 register space

The STK1160 exposes an 8-bit register space over USB control transfers.

### Vendor requests

| Operation       | `bmRequestType` | `bRequest` | `wValue`         | `wIndex` (register address) | `wLength` |
|-----------------|-----------------|------------|------------------|------------------------------|-----------|
| Register read   | `0xC0`          | `0x00`     | `0x0000`         | register address             | `1`       |
| Register write  | `0x40`          | `0x01`     | value (low byte) | register address             | `0`       |

The address space is 8-bit per register but groups of registers form 32-bit logical units. To access byte `n` of a 32-bit register based at address `X`, use address `X + n`.

### Address blocks

| Range      | Purpose                                          |
|------------|--------------------------------------------------|
| `0xx`      | GPIO                                             |
| `1xx`      | Video decoder configuration                      |
| `2xx`      | Serial bus (I²C master to SAA7113)               |
| `3xx`      | Timing generator (line/frame timing)             |
| `5xx`      | Audio                                            |

### Key registers (excerpt)

| Register | Name      | Notes                                                       |
|----------|-----------|-------------------------------------------------------------|
| `0x200`  | SICTL     | I²C-master control. Byte 0 of a 32-bit register.            |
| `0x201`  | SICTL+1   | Bit 0 = `RF` (read finished); bit 2 = `WF` (write finished).|
| `0x203`  | SICTL+3   | I²C slave address shifted left by 1 (write to address `addr << 1`). |
| `0x204`  | SBUSW     | Write subaddress (the I²C register on the target).          |
| `0x205`  | SBUSW+1   | Write data.                                                 |
| `0x206`  | SBUSW+2   | Write count (0 = 1 byte, default).                          |
| `0x208`  | SBUSR     | Read subaddress.                                            |
| `0x209`  | SBUSR+1   | Read data (valid after RF asserts).                         |

### Issuing an I²C write to the SAA7113

```
1. Write 0x203 = (saa7113_addr << 1)                  # set slave address
2. Write 0x204 = saa7113_reg                          # subaddress
3. Write 0x205 = data_byte                            # data
4. Write 0x200 = 0x01                                 # AI bit = "go"
5. Poll 0x201 until bit 2 (WF) is set                 # write finished
```

### Issuing an I²C read from the SAA7113

```
1. Write 0x203 = (saa7113_addr << 1)
2. Write 0x208 = saa7113_reg                          # subaddress
3. Write 0x200 = 0x20                                 # RD bit = read kick
4. Poll 0x201 until bit 0 (RF) is set                 # read finished
5. Read 0x209                                         # data
```

A short timeout (10–20 ms) on the poll is plenty; the chip processes a single byte in microseconds.

## SAA7113 video decoder

The SAA7113H is a public Philips/NXP analog video decoder. The chip's own datasheet is the authoritative reference; the subset most relevant to a TVM802 reimplementation is below.

### I²C slave address (RTS0 strap)

| Strap state   | Write addr | Read addr | 7-bit slave |
|---------------|------------|-----------|-------------|
| HIGH (default)| `0x4A`     | `0x4B`    | `0x25`      |
| LOW (3.3 kΩ pull-down) | `0x48` | `0x49` | `0x24`    |

The TVM802 board ships with RTS0 high (the datasheet default), so the slave is at `0x25` (7-bit). A reimplementation that wants to be defensive can probe both addresses.

### Tuning registers (subset)

| Reg  | Field                  | Default | Notes                                                                       |
|------|------------------------|---------|-----------------------------------------------------------------------------|
| `0x00` | Chip ident (RO)      | n/a     | Returns a chip ID byte; useful for "is this really an SAA7113?" probing.    |
| `0x02` | Analog input ctl 1   | varies  | `MODE[3:0]` selects which analog input pin is routed to the ADC.            |
| `0x03` | Analog input ctl 2   | varies  | `GAFIX` = manual-gain enable; `GAI18` / `GAI28` = gain MSBs for channels 1/2.|
| `0x04` | Analog input ctl 3   | varies  | `GAI17..GAI10` = manual gain low byte, channel 1.                           |
| `0x05` | Analog input ctl 4   | varies  | `GAI27..GAI20` = manual gain low byte, channel 2.                           |
| `0x0A` | Brightness           | `0x80`  | Unsigned 8-bit, midpoint = `0x80`.                                          |
| `0x0B` | Contrast             | `0x44`  | Signed 7-bit; `0x40` = unity, `0x44` ≈ 1.06×.                               |
| `0x0C` | Saturation           | `0x40`  | Signed 7-bit; `0x40` = 1.0×.                                                |
| `0x0D` | Hue                  | `0x00`  | Signed 8-bit; phase shift for chroma.                                       |
| `0x0E` | Chroma ctl 1         | varies  | `CSTD[2:0]` selects colour standard (NTSC/PAL/SECAM).                       |
| `0x0F` | Chroma gain ctl      | varies  | `ACGC` enables auto-chroma-gain; `CGAIN[6:0]` is the manual value.          |
| `0x1F` | Status (RO)          | n/a     | Lock bits, standard-detect bits.                                            |

Reserved (do not write): `0x14`, `0x18..0x1E`, `0x20..0x3F`, `0x63..0xFF`.

### Persistence

The SAA7113 retains its register values as long as USB Vbus is asserted. This means:

- A WinUSB tuning utility can set brightness/contrast/gain, exit, and the values **persist** for any subsequent driver (including the stock Windows driver re-binding).
- A full USB plug-cycle or host power-cycle resets to chip defaults — or to whatever the STK1160 loads from an attached serial EEPROM at init.

The operational pattern that works on this machine: a one-shot tuning script, run once at boot (or in a `.bat` on Windows startup), that writes the desired SAA7113 register set via the STK1160 vendor requests. Subsequent SurfaceMount / vision-app frames inherit the tuning.

## Pixel format and constraints

The USB stream is **8-bit UYVY 640×480**. Some implications:

- **No raw 10-bit access.** The SAA7113 has a 10-bit ADC but only 8 bits are exposed over USB. The 4× dynamic range win is invisible at the host. Improving image quality means improving the analog side (lighting, gain, cable length) — not chasing bit depth.
- **Mono on this machine.** Both cameras are monochrome; only the Y channel of UYVY carries useful information. U / V should be near `0x80` (neutral).
- **Interlaced source.** The SAA7113 deinterlaces in-chip; output frames are progressive at the line rate. A captured frame is one *field*, not a fused pair — see "Field recency" below.
- **No RGB.** The grabber's RGB format enumerates but `openStream` fails. Convert from UYVY in the host (Y plane → grey, or proper colourspace if needed).

## Pixel scale (mm per pixel)

Per-camera, per-axis scale factors live in the **controller** parameter store, NOT in the USB stream metadata:

| Camera | X scale (key) | Y scale (key) | Typical value                  |
|--------|---------------|---------------|--------------------------------|
| Up     | 20            | 21            | ~0.0446 / 0.0426 mm/pixel      |
| Down   | 34            | 35            | ~0.0268 / 0.0253 mm/pixel      |

Read with `read_param` (opcode `0x01`) at host startup and cache. **The X and Y scales differ by ~6 %** on the down camera — the effective pixels from the analog/interlaced chain are slightly non-square. Reimplementations should respect anisotropy when converting pixel measurements to millimetres.

## Mirror flags

Parameter key `58` packs camera mirror / orientation flags as `(up_mirror << 16) | (down_mirror & 0xFFFF)`. The values are encoded for the original vendor's vision DLL; a reimplementation can either honour them (read, apply post-capture flip) or replace with its own orientation-calibration step.

## Field recency

The SAA7113 deinterlaces by interleaving fields. When the camera mount is moving (e.g. mid-jog), consecutive fields capture different scenes — the deinterlaced "frame" then shows comb artefacts. For vision steps that require a single coherent capture:

1. Issue a `stop` (controller opcode `0x09`) on the camera-mount axis.
2. Wait for `status_poll.axis_state[i] == 0` AND `axis_position[i]` stable for two consecutive polls.
3. Then grab the next USB frame.

The settle window is dominated by the camera mount's own damping (typically 50–100 ms) plus one frame period for the in-flight USB buffer to flush. For static images (board fiducials, parts on a head not in motion) this is unnecessary; for moving captures (closed-loop vision during a jog or correction) it is essential.

## What the protocol does NOT expose

This is the controller-side protocol; it has no opinion on:

- Camera focus (mechanical adjustment on the lens; no electronic focus on either camera)
- White balance (mono cameras; trivial / ignored)
- Camera resolution beyond 640×480 (the chip's hard limit)
- Frame timing beyond the chip's NTSC/PAL clock (no master sync from the controller)
- The image-processing pipeline (host-side; the controller does not see pixel data)

For everything in this list, a reimplementation is on its own.
