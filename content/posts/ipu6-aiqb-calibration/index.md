---
title: "Extracting Sensor Calibration from Intel's AIQB Binary for libcamera"
date: 2026-05-07T21:00:00Z
draft: false
description: "Intel ships per-sensor calibration data in proprietary AIQB binaries alongside their Windows camera drivers. Here is how to parse them to generate libcamera Simple IPA tuning files with accurate CCMs and AWB limits."
ShowToc: true
ShowReadingTime: true
tags:
  - "linux"
  - "webcam"
  - "ipu6"
  - "libcamera"
  - "kernel"
  - "hardware"
  - "calibration"
categories:
  - "linux"
  - "hardware"
outputs:
  - "HTML"
---

If you followed the [Intel IPU6 webcam migration
post](/blog/posts/ipu6-webcam-libcamera-on-linux/), you know that
libcamera's Simple IPA falls back to `uncalibrated.yaml` when no
sensor-specific tuning file exists. That fallback enables AGC and AWB but
has no color correction matrix, which means colors depend entirely on the
grey world AWB algorithm to converge - and it often does not converge well,
especially under mixed or warm lighting.

Intel ships a per-sensor calibration binary with every Windows IPU6 camera
driver: a file with the `.aiqb` extension, sometimes called a CPFF. These
binaries contain the CCMs, AWB neutral locus, and sensor properties that
Intel's proprietary `icamerasrc` pipeline reads. The data was measured on
real hardware, which makes it more accurate than any generic default.

This post documents the AIQB binary format well enough to extract what
libcamera needs, and walks through the script that parses it. The OV2740
tuning file in libcamera (in review as of May 2026) was generated this way.

## Where AIQB files come from

There are two sources.

### Option A: intel/ipu6-camera-hal (easier)

Intel maintains a repository of AIQB calibration files for IPU6 sensors at
[intel/ipu6-camera-hal](https://github.com/intel/ipu6-camera-hal) under
`config/linux/ipu6ep/`. At the time of writing it covers:

```text
AR0234_TGL_10bits.aiqb
HI556_1BG502T3_ADL.aiqb
HI556_CJFLE25_ADL.aiqb
HM2170_1SG205N3_ADL.aiqb
HM2170_CJFME18_ADL.aiqb
OV02C10_1BG203N3_ADL.aiqb
OV02C10_1SG204N3_ADL.aiqb
OV02C10_CIFME14_ADL.aiqb
OV2311_ADL.aiqb
OV2740_CJFLE23_ADL.aiqb
ov01a10.aiqb
ov01a1s.aiqb
ov13b10.aiqb
ov8856.aiqb
```

Download whichever file matches your hardware:

```bash
gh api repos/intel/ipu6-camera-hal/contents/config/linux/ipu6ep \
  | python3 -c "import json,sys; [print(f['name']) for f in json.load(sys.stdin)]"

# Download a specific file
gh api repos/intel/ipu6-camera-hal/contents/config/linux/ipu6ep/OV2740_CJFLE23_ADL.aiqb \
  --jq '.download_url' | xargs curl -sL -o OV2740_CJFLE23_ADL.aiqb
```

If your sensor or module combination is not in that repository, use option B.

### Option B: OEM Windows driver installer

The authoritative source is the camera driver package your laptop
manufacturer ships for Windows. These are Inno Setup installers (.exe) that
bundle Intel's IPU6 driver, firmware, and the AIQB calibration files for
the specific camera module on that machine. This is how the OV2740
calibration for the ThinkPad X1 Carbon Gen 10 was obtained.

You need two tools: `p7zip` to unpack the outer installer archive, and
`innoextract` to unpack the Inno Setup payload.

```bash
# Arch Linux
sudo pacman -S p7zip innoextract
```

Download the Windows camera driver from your OEM support page (Lenovo,
Dell, HP, etc.) - look for "Intel IPU6 Camera Driver" or similar. For
the ThinkPad X1 Carbon Gen 10, the file is `n3ace31w.exe` from Lenovo's
support site. Then extract on Linux:

```bash
# Step 1: unpack the outer archive (some installers are just self-extracting zip)
7z x CameraDriver.exe -o camera-extracted/

# Step 2: find and unpack the Inno Setup payload
# The .exe itself or a file inside is usually an Inno Setup installer
innoextract camera-extracted/setup.exe -d camera-unpacked/

# Step 3: find the AIQB files
find camera-unpacked/ -name "*.aiqb"
```

The exact inner structure varies by OEM and driver version. For the Lenovo
ThinkPad X1 Carbon Gen 10 camera driver (`n3ace31w.exe`), the AIQB files
are nested inside a sub-installer that `7z` can also unpack directly:

```bash
7z x n3ace31w.exe -o ipu6-extracted/
find ipu6-extracted/ -name "*.aiqb"
# -> ipu6-extracted/.../OV2740_CJFLE23_ADL.aiqb
```

The naming convention is `SENSOR_MODULE_PLATFORM.aiqb` for Alder Lake
files (e.g. `CJFLE23` = Chicony camera module on Lenovo hardware). The
module name matters because the same sensor die is mounted with different
lenses and IR filters across products, and the calibration is
module-specific, not just sensor-specific.

## The AIQB binary format

AIQB is Intel's `ia_mkn` (Make Note) binary format, sometimes called CPFF
(Camera Parameter File Format). It is a flat sequence of tagged records
starting at offset `0x50` in the file. Each record starts with an 8-byte
header:

```text
struct ia_mkn_record_header {
    uint32_t size;      // total record size including this header
    uint8_t  fmt_id;    // format version
    uint8_t  key_id;    // key identifier
    uint16_t name_id;   // record type (the important field)
};
```

Records chain sequentially: `next_record = current_offset + size`. The
`name_id` field identifies the record type. The records we care about for
libcamera tuning:

| name_id | Name | Contents |
|---------|------|----------|
| 2 | CMC_GENERAL_DATA | Resolution, bit depth, Bayer order |
| 7 | CMC_SENSITIVITY | Base ISO |
| 18 | CMC_COLOR_MATRICES | Integer 3x3 CCMs per illuminant |
| 25 | CMC_ADV_COLOR_MATRICES | Float 3x3 CCMs with CCT in Kelvin |

Record id=25 (advanced color matrices) is present in all Alder Lake AIQB
files examined and is preferable: it stores the CCMs as single-precision
floats (rows summing to 1.0) and the color temperature directly in Kelvin,
so no integer scale factor needs to be guessed. Record id=18 uses integer
matrices with an unspecified scale that varies by file (typically 4096 or
8192).

### Record id=25 layout

After the 8-byte header:

```text
uint16_t num_light_srcs;
uint16_t num_sectors;
uint32_t hue_of_sectors[num_sectors];   // skip, for advanced segmented CCMs

// For each light source:
struct {
    uint32_t src_type;
    float    r_per_g;       // chromaticity at this illuminant
    float    b_per_g;
    float    cie_x;
    float    cie_y;
    uint32_t cct;           // color temperature in Kelvin
    float    traditional[9]; // 3x3 CCM, row-major, rows sum to 1.0
    float    advanced[num_sectors][9]; // per-sector CCMs, skip
} entries[num_light_srcs];
```

The `traditional` matrix is what libcamera's Ccm algorithm expects. The
`advanced` per-sector matrices implement a more complex spatially-varying
correction that the Simple IPA does not support.

## The parser script

The script `utils/tuning/parse_aiqb.py` in the libcamera tree (added in
the ov2740-tuning series, in review) walks the record chain, extracts the
CCMs and chromaticity locus, derives AWB max gains, and prints a
ready-to-use YAML block.

Basic usage:

```bash
python3 utils/tuning/parse_aiqb.py OV2740_CJFLE23_ADL.aiqb \
    --sensor-name ov2740 \
    --black-level 4096
```

The `--sensor-name` argument sets the output filename comment and defaults
to the sensor portion of the AIQB filename. The `--black-level` argument
takes the value in libcamera's 16-bit convention (where the effective 8-bit
black level is `value >> 8`). For the OV2740, the datasheet gives 0x40 at
10-bit (64 ADU), which is `64 << 6 = 4096` in 16-bit convention.

Example output for `OV2740_CJFLE23_ADL.aiqb`:

```text
File: OV2740_CJFLE23_ADL.aiqb (332756 bytes)
Records found: [1, 2, 3, 7, 9, 13, 15, 16, 17, 19, 20, 22, 25, 26, 28, 29, 31, 34, 37]

Sensor: 1932x1092, 10-bit, color_order=0
Base ISO: 53

Advanced color matrices (id=25, 8 entries, float CCMs):
  CCT=2319K  R/G=0.9898  B/G=0.3577
  CCT=2854K  R/G=0.7928  B/G=0.4199
  CCT=2884K  R/G=0.7145  B/G=0.4127
  CCT=3239K  R/G=0.6390  B/G=0.4478
  CCT=3865K  R/G=0.5599  B/G=0.5303
  CCT=4136K  R/G=0.5262  B/G=0.5276
  CCT=4939K  R/G=0.5101  B/G=0.6083
  CCT=6302K  R/G=0.4425  B/G=0.7123

Suggested AWB maxGainR=2.49, maxGainB=3.07
  (from min R/G=0.4425, min B/G=0.3577)
```

The AWB gain limits are computed from the minimum R/G and B/G
chromaticities across all illuminants, with 10% headroom:
`maxGainR = (1 / min_rg) * 1.1`. These are the values that prevent AWB
from overcorrecting toward extreme color temperatures that are outside the
sensor's calibrated range.

## Deriving the black level

The script does not extract the black level from the AIQB. The AIQB does
contain a black level record (id=3), but the format varies and the value
should be cross-checked against the sensor datasheet anyway. Typical
sources:

- Sensor datasheet (OV2740: register 0x4003 = 0x40, i.e. 64 ADU at 10-bit)
- Kernel driver: `ov2740.c` initializes `0x4003` to `0x40`
- The AIQB general data record (id=2) gives bit depth, which tells you
  how to interpret the register value

In libcamera's 16-bit convention: `blackLevel = (sensor_bl_adu << (16 - bit_depth))`.
For OV2740 at 10-bit: `64 << 6 = 4096`.

## What you get

The script produces a YAML block ready to drop into
`src/ipa/simple/data/<sensor>.yaml`. For the OV2740 with 8 illuminants
from 2319 K to 6302 K, the result is a 52-line tuning file with:

- BlackLevel at the correct value for BLC subtraction
- AWB gain limits derived from actual sensor chromaticity data
- 8 CCMs interpolated by libcamera's Ccm algorithm across the color
  temperature range

The color improvement over `uncalibrated.yaml` is visible: the uncalibrated
AWB converges to approximately R/G=0.91, B/G=0.90 (a ~9% green cast caused
by the AWB statistics bit-depth bug now fixed in master). With the tuning
file, it converges to near-neutral and the CCMs handle the remaining
per-illuminant correction.

## Limitations

This was reverse-engineered from a single AIQB file (OV2740_CJFLE23_ADL).
Known limitations:

- **Older AIQB format**: Files without record id=25 fall back to id=18
  (integer matrices). The scale autodetection works by checking which
  divisor produces row sums closest to 1.0, but it can fail if the file
  uses an unusual scale.
- **num_sectors > 0**: Some AIQB files may include per-sector advanced CCMs
  (the `advanced[num_sectors][9]` array). The script skips these correctly
  as long as `num_sectors` is parsed correctly from the header.
- **Black level record (id=3)**: Not parsed. Verify against the datasheet.
- **Different FIRST_RECORD_OFFSET**: The `0x50` offset was verified by hand
  for ipu6ep files. Older ipu6 files may differ.

If you run the script on a different AIQB and the CCMs show row sums far
from 1.0 or the chromaticity values are unreasonable (R/G outside 0.3-3.0),
something in the layout assumptions is off. The `Records found:` line and
a `xxd` inspection of the file around the flagged record are the starting
points for debugging.

## Next steps

If your sensor is in `intel/ipu6-camera-hal`, the path to a tuning file is
now straightforward:

1. Download the AIQB for your sensor/module combination
2. Run the parser with the correct `--black-level` for your sensor
3. Verify the output CCMs have row sums near 1.0 and chromaticities in range
4. Test with `LIBCAMERA_IPA_CONFIG_PATH` pointing to the output YAML
5. Submit the YAML as a patch to `src/ipa/simple/data/`

The parser script is included in the libcamera tree at
`utils/tuning/parse_aiqb.py` as part of the OV2740 tuning series.

## References

- [Intel IPU6 Webcam on Linux: From Proprietary Stack to Mainline](/blog/posts/ipu6-webcam-libcamera-on-linux/)
- [intel/ipu6-camera-hal](https://github.com/intel/ipu6-camera-hal) - AIQB files at `config/linux/ipu6ep/`
- [OV2740 tuning series](https://lists.libcamera.org/pipermail/libcamera-devel/2026-May/058643.html) - v1 in review on libcamera-devel
- [libcamera Simple IPA source](https://git.libcamera.org/libcamera/libcamera.git/tree/src/ipa/simple) - IPA algorithms and tuning YAML format
