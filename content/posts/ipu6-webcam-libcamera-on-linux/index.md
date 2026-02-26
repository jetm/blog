---
title: "Intel IPU6 Webcam on Linux: From Proprietary Stack to Mainline"
date: 2026-02-25T20:00:00Z
draft: false
description: "How to transition an Intel IPU6 webcam (OV2740) from the out-of-tree ipu6-drivers stack to mainline kernel + libcamera on Linux 6.19, with tuning fixes for the SoftISP color pipeline."
ShowToc: true
ShowReadingTime: true
tags:
  - "linux"
  - "webcam"
  - "ipu6"
  - "libcamera"
  - "kernel"
  - "arch-linux"
  - "hardware"
  - "pipewire"
categories:
  - "linux"
  - "hardware"
outputs:
  - "HTML"
---

The Intel IPU6 (Imaging Processing Unit, generation 6) is the camera subsystem in Tiger Lake, Alder Lake, Raptor Lake, and Meteor Lake laptops. If you have a recent ThinkPad, XPS, or Surface, your webcam likely runs through it. For years, getting it to work on Linux meant an out-of-tree driver stack from Intel's GitHub - four separate repositories, a DKMS module, proprietary firmware blobs, and a GStreamer-based relay daemon. It was fragile, broke on kernel updates, and couldn't survive suspend/resume on recent kernels.

The mainline kernel has had IPU6 ISYS support since 6.10. libcamera has had IPU6 support via its Simple pipeline handler since 0.3.2. But making the transition from the old stack to the new one isn't obvious - the pieces are spread across kernel modules, firmware, PipeWire configuration, and browser flags. This post documents the full migration on a ThinkPad X1 Carbon (Alder Lake) with an OmniVision OV2740 sensor, running CachyOS kernel 6.19.3.

## The Old Stack

The out-of-tree setup, typically installed via AUR packages like `archlinux-ipu6-webcam`, looks like this:

```
Sensor (OV2740)
  -> IPU6 ISYS (raw capture)
  -> IPU6 PSYS (hardware ISP)        [out-of-tree, not in mainline]
  -> intel-ipu6-camera-hal            [proprietary HAL]
  -> icamerasrc (GStreamer plugin)
  -> v4l2-relayd
  -> v4l2loopback (/dev/video0)
  -> Applications
```

This required six packages: `intel-ipu6-dkms-git`, `intel-ipu6ep-camera-bin`, `intel-ipu6ep-camera-hal-git`, `icamerasrc-git`, `v4l2-relayd`, and `v4l2loopback-dkms`. The DKMS modules had to be rebuilt for every kernel update, and Intel's out-of-tree repository stopped keeping up with kernel API changes around 6.16. Suspend/resume broke entirely - [intel/ipu6-drivers#381](https://github.com/intel/ipu6-drivers/issues/381) tracks the regression, which remains unfixed.

The PSYS module (Processing System - the hardware ISP) was never merged into mainline. Intel considers the hardware interface and ISP algorithms proprietary. Without PSYS, the entire `icamerasrc` pipeline can't function. On kernel 6.19, the DKMS modules simply won't build.

## The New Stack

The mainline path replaces the hardware ISP with libcamera's software ISP (SoftISP):

```
Sensor (OV2740)
  -> IPU6 ISYS (raw Bayer capture)    [in-tree since 6.10]
  -> libcamera Simple pipeline
  -> SoftISP (CPU/GPU debayering)
  -> PipeWire (Video/Source node)
  -> Applications
```

No DKMS, no proprietary blobs, no GStreamer relay chain. The trade-off is image quality - the SoftISP does CPU-based (or GPU-accelerated, since libcamera 0.7) Bayer demosaicing instead of the dedicated hardware ISP. Colors need tuning. But it works with mainline kernels, survives updates, and integrates natively with PipeWire.

## Step 1: Remove the Old Packages

Remove every component of the out-of-tree stack:

```bash
sudo pacman -Rns icamerasrc-git-fix \
  intel-ipu6ep-camera-hal-git-fix \
  intel-ipu6ep-camera-bin-fix \
  intel-ipu6-dkms-git-fix \
  intel-ivsc-firmware \
  v4l2-relayd \
  v4l2loopback-dkms-git-fix
```

The package names may differ depending on which AUR packages you installed. The `-Rns` flag removes dependencies and config files.

**Keep** `libcamera`, `libcamera-ipa`, `pipewire-libcamera`, and `gst-plugin-libcamera` - these are the new stack.

## Step 2: Verify Kernel Modules

After a reboot on kernel 6.10+, the in-tree modules should load automatically:

```bash
$ lsmod | grep -E "ipu6|ov2740|ivsc"
intel_ipu6_isys       151552  6
intel_ipu6             90112  7 intel_ipu6_isys
ipu_bridge             24576  2 intel_ipu6,intel_ipu6_isys
ivsc_csi               ...
ivsc_ace               ...
ov2740                 28672  0
```

The critical chain is: `mei_vsc_hw` -> `mei_vsc` -> `ivsc-ace` (powers the sensor) -> `ivsc-csi` (bridges CSI-2) -> `ov2740` (sensor driver) -> `intel_ipu6_isys` (capture). If `ivsc-ace` or `ivsc-csi` are missing, the sensor won't power on and libcamera will report "No sensor found."

The required kernel configs (check with `zgrep` on `/proc/config.gz`):

```
CONFIG_VIDEO_INTEL_IPU6=m
CONFIG_INTEL_VSC=m
CONFIG_INTEL_SKL_INT3472=m
```

CachyOS and recent Arch kernels have all three enabled as modules.

## Step 3: Verify libcamera Sees the Camera

```bash
$ cam --list
Available cameras:
1: Internal front camera (\_SB_.PC00.LNK1)
```

The warnings about "Unable to get rectangle" and "sensor kernel driver needs to be fixed" are cosmetic - the OV2740 kernel driver doesn't implement the full V4L2 selection API that libcamera prefers, but capture works regardless.

If `cam --list` shows nothing, check permissions. The user needs to be in the `video` group:

```bash
sudo usermod -aG video $USER
```

Log out and back in for the group change to take effect.

## Step 4: Fix the Color Pipeline

Out of the box, the SoftISP uses a generic uncalibrated profile. The image will have a heavy green tint and unstable brightness. libcamera logs confirm the fallback:

```
WARN: Configuration file 'ov2740.yaml' not found for IPA module 'simple',
      falling back to '/usr/share/libcamera/ipa/simple/uncalibrated.yaml'
```

The uncalibrated profile enables AGC (auto gain control) and AWB (auto white balance) but has no color correction matrix. The output has a visible green color cast. While the grey world AWB should theoretically compensate for any channel imbalance, the cast persists because of a bug in how the AWB interacts with the black level correction (details below). Without a CCM to compensate, everything looks green.

### The AGC flicker problem

The SoftISP's AGC algorithm (`src/ipa/simple/algorithms/agc.cpp`) uses histogram-based exposure and gain control. It computes a Mean Sample Value (MSV) from the luminance histogram and adjusts exposure/gain to converge on a target brightness. The problem is the control law: it's a bang-bang controller with a fixed ~10% step per frame.

```cpp
// libcamera 0.7.0 - agc.cpp:50-52
static constexpr uint8_t kExpDenominator = 10;
static constexpr uint8_t kExpNumeratorUp = kExpDenominator + 1;    // always +10%
static constexpr uint8_t kExpNumeratorDown = kExpDenominator - 1;  // always -10%
```

The hysteresis dead band is only +/-0.2 on a 0-5 MSV scale (+/-4%). When the correct exposure falls within one step of the current value - say, 5% above - the controller jumps +10%, overshoots, then -10%, undershoots, and oscillates forever. This is visible as brightness flicker on every sensor, but it's especially bad on the OV2740 behind IPU6 where the IVSC bridge adds latency to control application.

The fix is to replace the bang-bang controller with a proportional one, where the step size scales with the error magnitude:

```cpp
// Proportional: small error -> small step, large error -> large step
float step = std::clamp(static_cast<float>(error) * kExpProportionalGain,
                        -kExpMaxStep, kExpMaxStep);
float factor = 1.0f + step;
```

With `kExpProportionalGain = 0.04` and `kExpMaxStep = 0.15`:

| Error (MSV off target) | Step size |
|---|---|
| 2.5 (very dark) | +10% |
| 1.0 | +4% |
| 0.3 | +1.2% |
| 0.1 (nearly correct) | +0.4% |

The controller converges smoothly without overshooting. There's a ~3 second ramp-up from cold start (sensor defaults to 1x gain), but no flicker. This is a generic improvement that benefits all sensors on the Simple pipeline, not just the OV2740.

### Creating a tuning file

The OV2740 tuning file goes in `/usr/share/libcamera/ipa/simple/ov2740.yaml`. libcamera discovers it automatically by matching the sensor name. With the proportional AGC fix, AGC can remain enabled:

```yaml
# /usr/share/libcamera/ipa/simple/ov2740.yaml
# SPDX-License-Identifier: CC0-1.0
%YAML 1.1
---
version: 1
algorithms:
  - BlackLevel:
      blackLevel: 4096
  - Awb:
  - Ccm:
      ccms:
        - ct: 6500
          ccm: [ 2.49, -0.91, -0.26,
                -0.30,  1.20,  0.10,
                 0.07, -0.80,  2.19 ]
  - Adjust:
  - Agc:
```

The explicit `blackLevel: 4096` corresponds to the OV2740 kernel driver's BLC register value of 64 at 10-bit (register 0x4003 = 0x40 for the 180 MHz link frequency mode). At 16-bit representation: 64 * 64 = 4096. Without an explicit value, the auto-guess path in `blc.cpp` converges to the same effective value (16 at 8-bit), but specifying it explicitly documents the correct sensor characteristic.

### Why the green cast: AWB/BLC bit-depth mismatch

The CCM corrects a green cast caused by a bit-depth mismatch in the SoftISP's AWB algorithm. The statistics gathering code (`swstats_cpu.cpp`) accumulates raw pixel values at the sensor's native 10-bit depth (values 0-1023). But the AWB algorithm (`awb.cpp`) reads the black level as a `uint8_t` (value 16) and subtracts `blackLevel * nPixels` from the 10-bit sums. The correct per-pixel offset for 10-bit data is 64, not 16 - a 4x under-correction.

This causes the AWB grey world computation to underestimate the green channel's excess over red and blue. The resulting AWB gains are too low for full correction, leaving a ~9% green cast (R/G ≈ 0.91 without CCM).

The evidence: when the tuning file specifies `blackLevel: 0` (no subtraction), the AWB produces perfect ratios (R/G = 1.000, B/G = 1.001) because the offset mismatch disappears - but the shadows are lifted (minimum pixel values ~75 instead of 0). With the correct BLC of 4096 and no CCM, the green cast is clearly visible. The CCM compensates empirically until the AWB bit-depth handling is fixed upstream.

This bug affects all sensors with >8-bit output using the Simple pipeline. For 8-bit sensors, the AWB offset computation is correct.

The CCM matrix was computed iteratively - capturing frames, measuring RGB ratios in white regions, and adjusting until neutral grey actually read as neutral.

The measured progression across tuning iterations:

| Iteration | R/G ratio | B/G ratio | Ceiling luma | Notes |
|-----------|-----------|-----------|--------------|-------|
| uncalibrated.yaml | 0.819 | 0.773 | 88 | Heavy green, AGC flicker |
| CCM v1 (balanced rows) | 0.931 | 0.886 | 108 | Better, still green, dark |
| CCM v2 (measured correction) | 0.987 | 0.979 | 108 | Near-neutral, dark (no AGC) |
| CCM v2 + fixed gain | 1.023 | 1.086 | 169 | White reads as white |

A slight blue tint remains (B/G=1.086) and the left side of the frame is darker than the right due to lens shading - a per-pixel correction that the SoftISP doesn't implement. For a webcam, this is acceptable.

### Setting digital gain

The libcamera Simple IPA's AGC controls `exposure` and `analogue_gain` but not `digital_gain`. The sensor's V4L2 controls:

| Control | Range | Default | Managed by |
|---------|-------|---------|------------|
| exposure | 4-1102 | 1102 | libcamera AGC |
| analogue_gain | 128-1983 | 128 | libcamera AGC (128 = 1x, 1983 = 15.5x) |
| digital_gain | 1024-4095 | 1024 | Not managed (1024 = 1x) |

With the proportional AGC, exposure and analogue_gain are handled automatically. But the OV2740's analogue gain range (1x-15.5x) isn't always enough for indoor lighting. A 2x digital gain boost provides the missing headroom.

Since libcamera doesn't touch `digital_gain`, there's no race condition - a udev rule can set it at device probe time:

```bash
# /etc/udev/rules.d/99-ov2740-digital-gain.rules
SUBSYSTEM=="video4linux", KERNEL=="v4l-subdev*", ATTR{name}=="ov2740*", \
    ACTION=="add", RUN+="/usr/bin/v4l2-ctl -d /dev/$kernel --set-ctrl=digital_gain=2048"
```

No systemd services, no WirePlumber restart, no timing hacks. The gain is set once when the kernel creates the subdev node.

## Step 5: Browser Configuration

Chromium-based browsers (Chrome, Vivaldi, Edge) need the PipeWire camera feature flag to discover cameras through PipeWire instead of V4L2.

**Chrome** (`~/.config/chrome-flags.conf`):

```
--enable-features=WaylandWindowDecorations,WebRTCPipeWireCapturer,WebRtcPipeWireCamera
```

**Vivaldi** (`~/.config/vivaldi-stable.conf`):

```
--enable-features=UseOzonePlatform,WaylandWindowDecorations,WebRTCPipeWireCapturer,WebRtcPipeWireCamera
```

Important: multiple `--enable-features` lines don't merge - Chromium uses only the last one. Combine all features into a single comma-separated line.

**Firefox**: set `media.webrtc.camera.allow-pipewire` to `true` in `about:config`.

The feature flag name is `WebRtcPipeWireCamera` (not `PipeWireCamera` - check `strings /path/to/browser-binary | grep -i PipeWireCamera` to confirm your browser's naming).

## Suspend/Resume

The IPU6 firmware re-authentication fails after S3 resume on kernels 6.16+. After waking from suspend:

```
intel-ipu6: Unexpected magic number 0xffffffeb
intel-ipu6: FW authentication failed(-110)
```

The CSE (Converged Security Engine) can't re-authenticate the IPU6 firmware - the DMA buffer containing the firmware image isn't properly restored. This is tracked at [intel/ipu6-drivers#381](https://github.com/intel/ipu6-drivers/issues/381) and remains unfixed upstream.

The workaround is a systemd sleep hook that unloads and reloads the modules:

```bash
# /usr/lib/systemd/system-sleep/ipu6-suspend.sh
#!/bin/sh
case "$1" in
    pre)
        modprobe -r ov2740 intel_ipu6_isys intel_ipu6
        ;;
    post)
        modprobe intel_ipu6
        ;;
esac
```

This forces a clean re-probe (including fresh CSE authentication) on every resume.

## Upstreaming

Two patches have been submitted to the [libcamera mailing list](https://lists.libcamera.org/listinfo/libcamera-devel):

**1. [Proportional AGC for the Simple pipeline](https://lists.libcamera.org/pipermail/libcamera-devel/2026-February/057552.html).** The bang-bang controller in `src/ipa/simple/algorithms/agc.cpp` is replaced with a proportional controller. This is a generic improvement - any sensor on the Simple pipeline benefits from reduced overshoot. The patch is ~30 lines of changed logic.

**2. [OV2740 tuning file](https://lists.libcamera.org/pipermail/libcamera-devel/2026-February/057553.html).** The `ov2740.yaml` goes in `src/ipa/simple/data/`. As of libcamera 0.7, no sensor-specific tuning files exist for the Simple pipeline - only `uncalibrated.yaml`. This would be the first.

The [cover letter](https://lists.libcamera.org/pipermail/libcamera-devel/2026-February/057551.html) describes the series and testing methodology. During review, Milan Zamazal from Red Hat suggested investigating the black level as a potential root cause of the green cast. This led to discovering the AWB/BLC bit-depth mismatch described above - a systemic issue affecting all >8-bit sensors on the Simple pipeline. The v2 patches add an explicit black level and document the finding.

The CCM values are eyeball-calibrated from pixel measurements, not from a proper Macbeth ColorChecker calibration under controlled lighting. A proper submission would benefit from at least one calibration run, ideally at multiple color temperatures (2800K, 4000K, 6500K). The libcamera project provides tuning tools in `utils/tuning/` that could help.

The `digital_gain` udev rule is a system-level workaround that doesn't belong upstream. The proper fix would be extending the Simple IPA's AGC to also manage `V4L2_CID_DIGITAL_GAIN` when the sensor's analogue gain range is exhausted.

## Final Setup

After migration, the working configuration on a ThinkPad X1 Carbon (Alder Lake, OV2740, CachyOS 6.19.3, libcamera 0.7.0) is:

| Component | Source | Purpose |
|-----------|--------|---------|
| `intel-ipu6`, `intel-ipu6-isys` | In-tree kernel module | Raw Bayer capture |
| `ivsc-ace`, `ivsc-csi` | In-tree kernel module | Sensor power/CSI bridge |
| `ov2740` | In-tree kernel module | Sensor driver |
| `libcamera` 0.7.0 (patched) | Arch `extra` repo + AGC patch | Simple pipeline + proportional AGC |
| `pipewire-libcamera` | Arch `extra` repo | PipeWire integration |
| `ov2740.yaml` | Patched into libcamera | BLC + CCM + AGC |
| `99-ov2740-digital-gain.rules` | udev rule | digital_gain=2x at probe |

Image quality is noticeably worse than the proprietary Intel stack - the SoftISP doesn't have per-pixel lens shading correction, the CCM is a single global matrix rather than a per-channel curve, and the debayer algorithm is simpler. But it works with mainline kernels, doesn't break on updates, integrates natively with PipeWire, and requires zero out-of-tree code.

For video conferencing, which is what most laptop webcams are used for, it's sufficient.

## References

- [intel/ipu6-drivers](https://github.com/intel/ipu6-drivers) - Intel's out-of-tree IPU6 driver repository
- [intel/ipu6-drivers#381](https://github.com/intel/ipu6-drivers/issues/381) - Suspend regression on kernel 6.16+
- [libcamera IPU6 support](https://patchwork.libcamera.org/patch/19967/) - Simple pipeline handler for intel-ipu6
- [libcamera 0.7.0 release](https://lists.libcamera.org/pipermail/libcamera-devel/) - GPU-accelerated SoftISP
- [Arch Linux Forums - IPU6 webcam](https://bbs.archlinux.org/viewtopic.php?pid=2287604) - Community OV2740 tuning discussion
- [archlinux-ipu6-webcam](https://github.com/stefanpartheym/archlinux-ipu6-webcam) - AUR package for old stack (with suspend workaround scripts)
- [Arch Wiki - Libcamera](https://wiki.archlinux.org/title/Libcamera)
- [ov2740.yaml tuning file](https://lists.libcamera.org/pipermail/libcamera-devel/2026-February/057553.html) - Tuning file from this post (submitted upstream)
