# EVKphone — Event Camera Mobile Acquisition App · User Guide

EVKphone is an Android app that connects Prophesee EVK5-series event cameras
(EVK3/EVK4 also supported) directly to your phone — **no root required** — for
live event preview, synchronized multi-modal recording, and playback.

---
[中文文档 (Chinese)](README_CN.md)
---

## 1. Key Features

### Live Event Preview
The moment a camera is connected you see the event stream: **red = ON events,
blue = OFF events** on a black background, with old traces fading out.
Both orientations are supported (landscape uses a two-pane layout with the
preview dominating); rotating the phone **does not interrupt** the stream.

### Synchronized 3-Modal Recording
One recording produces four files sharing the same name prefix:

| File | Content |
|---|---|
| `rec_<time>.raw` | Event-camera data (standard EVT3; opens directly in Metavision software on PC) |
| `rec_<time>_imu.csv` | Phone IMU data (accelerometer + gyroscope + rotation vector, ~990 Hz, hardware timestamps) |
| `rec_<time>.mp4` | Phone rear-camera video (1080p 30 fps, no audio) |
| `rec_<time>_metadata.json` | Time-synchronization information (see Section 4) |

IMU and RGB video are optional modalities you can toggle before each
recording; IMU is on by default.

### Event File Playback
Put `.raw` files into `/sdcard/evk_data/` on the phone and the app plays them
back **smoothly at exactly 1× real time**, with pause / resume / replay /
progress-bar seeking. Files recorded by this app and by Metavision on PC both
work.

### Hot-Pixel Masking (sensor level)
Faulty/hot pixels are blocked **inside the sensor itself** — masked pixels
never generate data, so recordings, preview, and statistics are clean at the
source (this is not post-hoc filtering).

- 3 measured hot-pixel coordinates are built in and enabled out of the box;
- You can add coordinates (`x,y`), delete individual points, restore the
  defaults, or clear the list;
- Changes take effect immediately and are saved automatically. When switching
  to a different camera body, re-derive the hot-pixel coordinates and edit the
  list accordingly.

### RGB Zoom
Pinch on the preview or drag the slider for digital zoom (magnification shown
live). Zoom applies to both the preview and the recorded video. After changing
the event camera lens, use zoom to align the RGB and event-camera fields of
view.

### Event-Rate Control (ERC)
Four presets: 2M / 10M / 30M / 100M ev/s. On phones with a **USB 2** port keep
ERC **enabled** (default 10M), or data overflows in dynamic scenes; USB 3
phones can raise it or turn it off.

---

## 2. Requirements

| Item | Requirement |
|---|---|
| Phone | 64-bit ARM Android device (arm64-v8a), **Android 11 or newer** |
| Port | USB OTG/Host (for the camera); not needed for file playback only |
| Camera | Prophesee EVK series (EVK2/EVK3/EVK4/EVK5, via the Treuzell board protocol) |
| Permissions | "All files access" (for the evk_data folder); camera permission (RGB modality only) |

**Connection tip**: connecting the camera directly to the phone often fails to
enumerate — use a **USB 3 hub / dock** (the tested, stable setup). If the
camera repeatedly disconnects and re-appears, check the dock's power supply
first. On USB 2 phones, keep ERC enabled.

---

## 3. Quick Start

1. **Install** the APK (allow "install from unknown sources" when prompted);
2. **Grant permissions**: on first launch tap the license row on the status
   card → "Grant file access" → allow in system settings;
3. **Connect the camera**: plug in via OTG cable/dock → the camera appears in
   the "USB devices" card → tap "Connect VID xxxx" → grant USB access → the
   status shows "streaming" and the preview comes alive;
4. **Record**: choose modalities (IMU / camera) in "Capture controls" → tap
   "Start recording" → "Stop recording";
5. **Get your files**: open `/sdcard/evk_data/` in any file manager or via MTP
   from a PC — all recordings are there;
6. **Play back**: in the "Event files" card, tap "Play" on any `.raw`.

---

## 4. Data Files & Time Synchronization

The three data streams are aligned through anchors in `metadata.json`
(the same scheme as the PC-side recording script, so existing processing
pipelines work unchanged):

- `time_base.unix_minus_monotonic_ns` — fixed offset between the phone's
  boot clock and Unix time;
- `event_camera.record_start_ns / record_start_monotonic_ns` — recording
  start in both time bases;
- `event_camera.first_event_ts_us / last_event_ts_us` — first/last event
  timestamps on the event clock (event-level precision);
- `event_camera.masked_hotpixels` — hot pixels masked during this recording;
- `imu.samples / estimated_rate_hz` — IMU sample count and measured rate;
- `video.start_monotonic_ns / duration_ns` — video start and duration.

**Alignment method**: IMU and video already live on the phone clock; the event
clock maps linearly onto it via `record_start + first_event_ts_us`. The IMU
CSV column names match the PC script `record_event_imu_session.py` exactly
(one extra `sensor` column notes each row's source), so your existing
processing scripts need no changes.

---

## 5. Licensing & Activation 

- **14-day full-feature trial**: starts automatically on first launch;
  activate any time, before or after it expires;


---

## 6. FAQ

**Q: The camera doesn't show up in the USB device list?**
Make sure OTG is enabled and the dock is powered; try a USB 3 hub (direct
connection succeeds rarely).

**Q: The camera appears then disappears repeatedly?**
Insufficient power or a loose connection. Check the dock's independent power
supply; if the camera stops responding entirely, power-cycle it by
unplugging/replugging.

**Q: What to watch out for when installing an update?**
**Always tap "Stop & disconnect camera" inside the app before installing.**
Killing the app while the camera is streaming can hang the camera firmware
(it stops responding until physically power-cycled).

**Q: Recordings are huge.**
At 10M ev/s expect ~37 MB/s (≈2.2 GB per minute) — that's normal; lower the
ERC preset to reduce data volume.

**Q: IMU rate is only 200 Hz?**
Some phone builds cap sensor rates. The app declares the high-sampling-rate
permission; most devices reach ~990 Hz.

**Q: The RGB video isn't 1080p?**
On some phones the camera falls back automatically; trust the actual video
file (a note field in the metadata says the same).

---

## 7. Copyright & Open-Source Licenses

© 2026 the author. All rights reserved. Redistribution or modification
without permission is prohibited.

Open-source components bundled with the app (with thanks):

| Component | License | Role |
|---|---|---|
| [OpenEB](https://github.com/prophesee/openeb) 5.2.0 | Apache License 2.0 | Event-camera HAL/driver layer |
| [libusb](https://libusb.info) 1.0.30 | GNU LGPL 2.1 | USB communication (dynamically linked) |
| AndroidX / CameraX / Jetpack Compose | Apache License 2.0 | Google UI/camera frameworks |

Full license texts are included in the corresponding directories of the
project.
