# Camera mux

The TVM802 ships with two cameras (top-down board fiducial, bottom-up part vision) that share **one** physical capture device. They are routed through a CD4052 analog multiplexer, switched by the controller's digital outputs.

This means the camera-select signal is sent over the same TCP protocol as motion, via output bit toggles.

## Output bits involved

| Mask     | ID              | Role in camera mux                                        |
|----------|-----------------|-----------------------------------------------------------|
| `0x8000` | `cam_mux`       | Mux relay select line. Set = up camera; clear = down.     |
| `0x0200` | `lamp_enable`   | Light-ring power.                                         |
| `0x0001` | `cam_light_pair`| Camera-light pairing enable.                              |

The light ring is paired with the active camera. To get a useful image with light, the host writes a composite output word that includes both the mux selection and the lamp bits.

## Common composite words

| Word     | Effect                                              |
|----------|-----------------------------------------------------|
| `0x0000` | Down camera, light off                              |
| `0x0201` | Down camera, light on (`lamp_enable | cam_light_pair`) |
| `0x8200` | Up camera, light on (`cam_mux | lamp_enable`)       |
| `0x8201` | Up camera, light on with pairing                    |

## Selecting a camera

Direct ON / OFF of the bit pair:

```
Select DOWN cam (with light):
    0x14 output_bit_on_immediate  bit = 0x0201   # lamp + pair
    0x15 output_bit_off_immediate bit = 0x8000   # cam_mux off

Select UP cam (with light):
    0x14 output_bit_on_immediate  bit = 0x8200   # cam_mux + lamp
    0x15 output_bit_off_immediate bit = 0x0001   # pair off (optional)
```

A single `set_outputs_immediate` (opcode `0x04`) with the full word is equivalent and atomic; preferred for camera switching since it avoids a transient "neither" state.

## Settle time

After a mux switch, allow at least **one full frame** (typically 33 ms at 30 fps) before sampling pixels. The CD4052 itself settles in microseconds, but the host's USB grabber's internal pipeline lags by 1-2 frames.

A safer approach: switch, wait 50-100 ms, then drain the camera's recent frames and read the next one. This avoids "wrong camera" frames sneaking into a vision step.

## Reimplementing in OpenPnP

The standard OpenPnP `SwitcherCamera` pattern fits exactly:

- One physical `OpenPnpCaptureCamera` device for the underlying capture.
- Two virtual `SwitcherCamera` instances ("Top Camera", "Bottom Camera") referencing it with mux values `0` (down) and `1` (up).
- A `ReferenceActuator` that drives the mux: actuation value `0` writes the down-camera composite output word; value `1` writes the up-camera word.

## DirectShow caveats (legacy)

The stock host uses a hand-rolled DirectShow filter to drive the grabber. It does not switch via DirectShow crossbar — it uses controller output bits. A reimplementation should do the same: do NOT look for an in-band camera-select API; there isn't one on the USB side.
