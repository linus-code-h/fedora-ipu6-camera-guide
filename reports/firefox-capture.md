# Firefox camera instability on Fedora IPU6 setup, with libcamera crash and PipeWire EBUSY logs

Intended destination: Mozilla `Core :: WebRTC: Audio/Video` for initial triage, potentially redirected to PipeWire/libcamera. This report is not yet submitted to an upstream tracker. It does not establish a Firefox-only defect.

## Environment

- Dell Latitude 9430, Intel Alder Lake IPU6, OV02C10 behind IVSC.
- Fedora 44 Workstation, kernel `7.2.7-200.fc44.x86_64`.
- Firefox `156.0.1-1.fc44`.
- PipeWire `1.6.9-1.fc44`, WirePlumber `0.5.17-1.fc44`, libcamera `0.7.1-1.fc44`.
- Intel proprietary HAL / `icamerasrc` / `v4l2-relayd` / v4l2loopback also installed.
- External Microsoft LifeCam connected as an additional camera.

## Observations

After recovering a separate relay-startup failure, the camera app and Zoom using **Intel MIPI Camera** worked. The owner reported flickering and unstable behavior in Firefox.

Fedora's installed Firefox defaults set:

```javascript
pref("media.webrtc.camera.allow-pipewire", true);
```

The profile contained only a similarly spelled but ineffective user preference, `media.webrtc.camera.allow_pipewire=false` (underscore). This was a local configuration mistake; its origin is unknown, and it is not claimed as a Firefox bug.

During the investigation, the user-session journal showed:

```text
09:55:59 FATAL default pipeline_handler.cpp:398 assertion "data->queuedRequests_.empty()" failed in stop()
09:56:02 wireplumber.service: Failed with result 'core-dump'.
```

After restarting, libcamera logged that `ov02c10.yaml` was missing and that it was using `uncalibrated.yaml`; it also reported missing sensor-helper/static-property information.

At 09:57, PipeWire repeatedly failed `VIDIOC_S_FMT` for the virtual MIPI device with errno 16 (`Device or resource busy`) and failed format negotiation. Negotiation described NV12, 1280×720, 30/1 fps.

The test site was https://de.webcamtests.com/ . In a follow-up, the owner confirmed that Zoom and the website were briefly open at the same time and that camera access worked in only one of them at a time. The owner now reports normal operation.

This makes capture contention a plausible explanation for the EBUSY messages; those messages alone are not evidence of a Firefox defect. It does not establish that concurrent access caused the flicker or the libcamera crash. The exact capture source and timing relative to each logged error remain unconfirmed.

## Workaround and result

A new profile `user.js` was created with:

```javascript
user_pref("media.webrtc.camera.allow-pipewire", false);
```

The owner was asked to close other previews, fully restart Firefox and select **Intel MIPI Camera**. The owner subsequently confirmed that everything worked.

The browser's effective runtime preference was not independently read. A later on-disk check showed the correct override in `user.js` while `prefs.js` still contained only the old underscore key. The exact restart sequence and selected source were not separately confirmed. Consequently, this is a successful recovery report with a plausible backend workaround, not an isolated A/B reproduction.

## Expected behavior and next diagnostic steps

The selected camera should deliver a stable preview. Switching or stopping capture should not crash the session manager. Busy devices should produce a clear error and recover when released.

For further investigation, confirm the selected node, read the effective preference, test one capture client at a time on the named site, and correlate Firefox/PipeWire/libcamera logs while switching backends. This has not been done yet, and the working machine has not been deliberately regressed to collect it. A new Firefox-specific defect should not be asserted from the busy-device messages without this isolation.

Related context, not established duplicates:

- https://bugzilla.mozilla.org/show_bug.cgi?id=1945597
- https://bugzilla.mozilla.org/show_bug.cgi?id=1946916 — `RESOLVED WORKSFORME`, primarily virtual-camera enumeration.

Prepared with an AI assistant from local diagnostics and owner reports. No images, full browser profile, credentials or private host/profile paths are included.
