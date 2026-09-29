# Fedora Intel IPU6 camera: recovery notes for a Dell Latitude 9430

[Deutsche Kurzanleitung](README.de.md)

A documented case of an Intel MIPI camera failing after a Fedora kernel update, followed by Firefox camera instability. The owner confirmed that the camera app, Zoom and Firefox worked again after the recovery steps below.

These are observed workarounds for one machine, not an upstream driver fix or a universal installation guide. No packages need to be removed. Device numbers can change between boots.

## Environment observed on 2026-09-29

| Component | Version / hardware |
| --- | --- |
| Laptop | Dell Latitude 9430 |
| Distribution | Fedora Linux 44 Workstation |
| Kernel | `7.2.7-200.fc44.x86_64` |
| Camera | Intel Alder Lake IPU6 (`8086:465d`), OV02C10 behind IVSC |
| Firefox | `156.0.1-1.fc44` |
| PipeWire | `1.6.9-1.fc44` |
| WirePlumber | `0.5.17-1.fc44` |
| libcamera | `0.7.1-1.fc44` |
| v4l2-relayd | `0.2.0-2.20251028gitd6ec36a.fc44` (RPM Fusion) |
| akmods | `0.6.2-14.fc44` (Fedora) |
| Intel IPU6 akmod | `0.0-25.20250909git4bb5b4d.fc44` |
| v4l2loopback | `0.15.4-1.fc44` |

The installed camera stack includes `ipu6-camera-hal`, `ipu6-camera-bins`, `gstreamer1-plugins-icamerasrc`, `v4l2-relayd` and `v4l2loopback`. An external Microsoft LifeCam was also connected.

## 1. Camera relay fails during the first boot of a new kernel

The sensor was present, but the relay started before the additional kernel modules finished building:

| Local journal time | Event |
| --- | --- |
| 09:44:23 | akmods began building missing modules |
| 09:44:26 | relay exhausted its restart limit; no virtual camera device yet |
| 09:44:46 | Intel IPU6 module build/install completed successfully |
| 09:44:55 | v4l2loopback build/install completed; module loaded |

The relay stayed in `failed (start-limit-hit)`. Its log included:

```text
grep: /sys/devices/virtual/video4linux/*/name: No such file or directory
Start request repeated too quickly.
Failed with result 'start-limit-hit'.
```

The first line above is translated from the German journal. The virtual device subsequently existed, but `intel_ipu6_psys` was not loaded.

### Check before changing anything

```bash
uname -r
systemctl status akmods.service v4l2-relayd@icamerasrc.service --no-pager
journalctl -b -u akmods.service -u v4l2-relayd@icamerasrc.service --no-pager
modinfo -n intel_ipu6_psys
modinfo -n v4l2loopback
v4l2-ctl --list-devices
```

Only use the following recovery if the modules for the running kernel have finished building successfully and the named camera relay is already part of your installed setup. A build failure is a different problem.

### Recovery used on this machine

Close camera previews, then run:

```bash
sudo modprobe intel_ipu6_psys &&
sudo systemctl reset-failed v4l2-relayd@icamerasrc.service &&
sudo systemctl restart v4l2-relayd@icamerasrc.service
```

Verify:

```bash
systemctl show v4l2-relayd@icamerasrc.service \
  -p ActiveState -p SubState -p NRestarts
v4l2-ctl --list-devices
```

Observed result: `active`, `running`, `NRestarts=0`. The **Intel MIPI Camera** device changed from output-only to a capture device configured for NV12, 1280×720 at 30 fps. The owner confirmed working video in the camera app and Zoom.

This did not install a persistent boot-order correction. Recovery after another kernel update has not been tested. See the [relay report](reports/relay-startup.md) for the proposed area to investigate.

## 2. Firefox flickering while Zoom works

During the Firefox investigation, the session logs showed:

```text
FATAL default pipeline_handler.cpp:398 assertion "data->queuedRequests_.empty()" failed in stop()
wireplumber.service: Failed with result 'core-dump'.
```

PipeWire also logged repeated `VIDIOC_S_FMT` failures with errno 16 (`Device or resource busy`) for the virtual MIPI camera. libcamera used `uncalibrated.yaml` because an OV02C10-specific tuning file was unavailable.

The owner later identified the test site as https://de.webcamtests.com/ and confirmed that Zoom and the site were briefly open simultaneously; only one could access the camera at a time. This makes contention a plausible explanation for the busy-device messages. The owner now reports normal operation. The flicker and libcamera crash were not isolated to a specific application, capture source or concurrent-access sequence.

### Firefox workaround

1. Close other camera previews for the comparison.
2. Open `about:config` in Firefox.
3. Find the exact existing preference **`media.webrtc.camera.allow-pipewire`** and set it to **`false`**.
4. Fully quit and reopen Firefox.
5. Select **Intel MIPI Camera** on the website and allow camera access normally.

The spelling matters: `allow-pipewire` has a **hyphen**. A manually created `allow_pipewire` preference with an underscore does not change this setting.

On this machine the setting was supplied through the active profile's `user.js`:

```javascript
user_pref("media.webrtc.camera.allow-pipewire", false);
```

The owner subsequently reported that everything worked. The browser's runtime preference was not independently inspected, and a controlled test switching repeatedly between backends was not performed. Treat this as a successful recovery report, not proof of a particular upstream defect.

To undo the workaround, remove this override from `user.js` if you added it there, then reset the exact preference in `about:config`. Removing `user.js` alone does not reset a preference already saved by Firefox.

## Earlier, separate kernel problem

With kernel `7.2.5`, the sensor did not bind and the journal contained `mei-csi probed without device fwnode!`. Older `7.1.13` logs showed sensor binding. With `7.2.7`, sensor binding was present again. Do not assume that every later camera failure is the same kernel regression.

Related references:

- [Linux IPU bridge regression report](https://lists.openwall.net/linux-kernel/2026/09/01/3091) — a different Dell model with a matching historical error.
- [Mozilla bug 1945597](https://bugzilla.mozilla.org/show_bug.cgi?id=1945597) — discusses selecting the V4L2 backend.
- [Mozilla bug 1946916](https://bugzilla.mozilla.org/show_bug.cgi?id=1946916) — related virtual-camera enumeration issue, marked `RESOLVED WORKSFORME` when checked; not established as a duplicate of this case.

## Reports and remaining work

- [Relay startup report](reports/relay-startup.md): candidate for the RPM Fusion `v4l2-relayd` package maintainers.
- [Firefox/capture-stack report](reports/firefox-capture.md): candidate for Mozilla WebRTC triage, with possible follow-up in PipeWire/libcamera.

The report files are prepared for upstream submission. Their presence in this repository does **not** mean an upstream tracker has received them. Submission status and links belong in [PUBLISHING.md](PUBLISHING.md).

No camera images, credentials, complete browser profiles or personal host/profile paths are included. This guide was prepared with an AI assistant from local diagnostic output; observed facts, owner confirmations and untested hypotheses are distinguished above.
