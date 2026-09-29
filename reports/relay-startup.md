# v4l2-relayd remains failed after akmods finishes camera modules on first boot of a new kernel

Intended destination: RPM Fusion, `v4l2-relayd` package maintainers. This report is not yet submitted to their tracker.

## Environment

- Dell Latitude 9430, Fedora 44 Workstation.
- Kernel `7.2.7-200.fc44.x86_64`, installed shortly before the observed boot.
- Intel Alder Lake IPU6 (`8086:465d`), OV02C10/IVSC.
- `v4l2-relayd-0.2.0-2.20251028gitd6ec36a.fc44` from RPM Fusion.
- `akmods-0.6.2-14.fc44` from Fedora.
- `akmod-intel-ipu6-0.0-25.20250909git4bb5b4d.fc44`.
- `akmod-v4l2loopback-0.15.4-1.fc44`.

## Observed trigger and result

On the first observed boot after installing the new kernel, missing camera modules were built during startup. The already configured `v4l2-relayd@icamerasrc.service` started before the virtual camera existed, restarted repeatedly, and reached `start-limit-hit` before module installation completed.

Relevant local journal sequence on 2026-09-29:

```text
09:44:23 akmods.service starts
09:44:25 relay cannot find /sys/devices/virtual/video4linux/*/name
09:44:26 relay: Start request repeated too quickly
09:44:26 relay: Failed with result 'start-limit-hit'
09:44:46 Building and installing intel-ipu6-kmod [ OK ]
09:44:55 Building and installing v4l2loopback-kmod [ OK ]
09:44:55 systemd-modules-load: Inserted module 'v4l2loopback'
```

This is a shortened transcription; the original device-not-found message was in German. Both module builds succeeded. At inspection, the virtual camera existed but the relay was still failed and `intel_ipu6_psys` was not loaded.

The installed relay template included:

```ini
[Unit]
PartOf=v4l2-relayd.service
After=modprobe@v4l2loopback.service systemd-logind.service

[Service]
Restart=always
```

Its `ExecStart` locates a device by matching `CARD_LABEL` against `/sys/devices/virtual/video4linux/*/name`.

## Expected behavior

If installed camera modules are being built successfully during startup, the configured relay should start after its required modules/devices become available, or recover automatically after their installation. It should not remain failed for the session solely because they were temporarily absent.

## Recovery and validation

After both module builds completed, the owner ran:

```bash
sudo modprobe intel_ipu6_psys &&
sudo systemctl reset-failed v4l2-relayd@icamerasrc.service &&
sudo systemctl restart v4l2-relayd@icamerasrc.service
```

The relay then showed `active/running`, `NRestarts=0`. The virtual camera offered Video Capture, NV12, 1280×720 at a configured 30 fps. The owner confirmed working camera-app and Zoom previews.

## Scope and proposed investigation

Investigate ordering/device readiness and recovery after akmods completes, including loading PSYS where required. Merely adding `After=akmods.service` may not be sufficient: activation dependencies, device creation and module loading also need review. No systemd patch has been tested or proposed as a proven fix.

This sequence was observed once for this kernel. It has not been reproduced with deliberately removed modules, and no modules were removed during diagnosis. This is separate from an earlier sensor-binding failure under kernel 7.2.5.

Prepared with an AI assistant from local diagnostics; the owner authorized publication and confirmed the application-level recovery. Full logs, private profile data and camera images are not included.
