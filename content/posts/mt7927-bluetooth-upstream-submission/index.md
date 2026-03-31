---
title: "MT7927 Bluetooth: From DKMS to Upstream"
date: 2026-03-05T10:00:00Z
draft: false
description: "MT7927 Bluetooth driver patches merged into bluetooth-next - the full journey from DKMS package to upstream acceptance, including 5 revision cycles, automated review tools, and community testing across 16+ boards."
ShowToc: true
ShowReadingTime: true
tags:
  - "linux"
  - "bluetooth"
  - "kernel"
  - "mediatek"
  - "mt7927"
  - "upstream"
  - "mailing-list"
categories:
  - "linux"
  - "hardware"
outputs:
  - "HTML"
---

In [Part 1]({{< ref "/posts/enabling-mt7927-bluetooth-on-linux" >}}), I documented getting MT7927 Bluetooth working through a DKMS package - patching three missing layers (USB device ID, hardware variant support, and firmware) into an out-of-tree build. That post ended with:

> As of February 2026, none of the three layers have reached mainline Linux.

This post covers what happened next: submitting all three layers upstream and getting the BT driver patches merged after five revision cycles.

**Update (2026-03-31):** The BT driver patches have been [merged into bluetooth-next](https://lore.kernel.org/linux-bluetooth/20260330-mt7927-bt-support-v4-0-cecc025e7062@jetm.me/) by Luiz Augusto von Dentz. They will ship in mainline Linux 7.1 or 7.2.

## Two Submissions, Two Channels

MT7927 Bluetooth support requires changes in two separate kernel repositories, each with its own submission process:

| Component | Repository | Channel | Submission |
|-----------|-----------|---------|------------|
| BT firmware | linux-firmware | GitLab MR | [MR !946](https://gitlab.com/kernel-firmware/linux-firmware/-/merge_requests/946) |
| BT driver (btusb + btmtk) | linux kernel | linux-bluetooth@ | [Lore thread](https://lore.kernel.org/linux-bluetooth/177272816248.352280.12453518046823439297@jetm.me/) |

Each moves independently. The firmware can land before the driver patches. Users need both, but the kernel doesn't care about ordering.

## Bluetooth Firmware: linux-firmware MR !946

The simplest of the two. `linux-firmware` is a git repository of binary firmware blobs, hosted on [kernel-firmware GitLab](https://gitlab.com/kernel-firmware/linux-firmware). Submissions are merge requests, not mailing list patches.

The MT6639 Bluetooth firmware (`BT_RAM_CODE_MT6639_2_1_hdr.bin`, 688 KB) was extracted from ASUS's publicly available Windows WiFi driver package (V5603998, compile date 2025-06-06). The firmware lives inside `mtkwlan.dat` - a WiFi firmware container - not distributed as a separate Bluetooth file. An `extract_firmware.py` script parses the binary container, locates the embedded BT firmware by its filename string, and writes the raw blob.

The MR adds the firmware file under `mediatek/mt6639/` with the WHENCE entry documenting its origin. The CI pipeline passes and the MR shows `can_be_merged`. No reviewer comments yet.

## Bluetooth Driver: linux-bluetooth

The BT driver changes grew from two patches to eight over five revisions:

1. **btmtk**: Add MT6639 (MT7927) hardware variant support - firmware naming, section filtering, CHIPID workaround, initialization sequence
2. **btmtk**: Fix ISO interface setup for devices with a single alternate setting
3. **btusb**: Six per-device USB ID commits, each with `Tested-by` trailers from community testers

### The Section Filter

The section filtering in patch 1 is the critical piece. The MT6639 firmware contains 9 sections, but only 5 are Bluetooth. The remaining sections are WiFi/other subsystem data. Without filtering by `dlmodecrctype & 0xFF == 0x01`, the driver sends non-Bluetooth sections to the chip, causing an irreversible subsystem hang. This matches the behavior of MediaTek's Windows driver.

A v2.1-20 bug fix caught a critical regression in this filter: the `dlmodecrctype` check wasn't gated on `dev_id == 0x6639`, meaning it would apply to all MediaTek Bluetooth chips - potentially breaking MT7921, MT7925, and every other chip that goes through the same firmware loading path. The upstream patch correctly scopes the filter to MT6639 only.

### Confirmed Hardware

Patches 3-8 add USB IDs confirmed by community testing across multiple hardware platforms. Each ID has its own commit with `Tested-by` trailers:

| USB ID | Hardware | Tested by |
|--------|----------|-----------|
| `0489:e13a` | ASUS ROG Crosshair X870E Hero | Jose Tiburcio Ribeiro Netto |
| `0489:e0fa` | Lenovo Legion Pro 7 16ARX9 | Llewellyn Curran |
| `0489:e10f` | Gigabyte Z790 AORUS MASTER X | Chapuis Dario, Evgeny Kapusta |
| `0489:e110` | MSI X870E Ace Max | Nitin Gurram |
| `0489:e116` | TP-Link Archer TBE550E | Thibaut Francois |
| `13d3:3588` | ASUS X870E-E / ProArt X870E-Creator | Jose Tiburcio Ribeiro Netto, Ivan Lubnin |

New IDs can be added incrementally as users report them. The current set covers all known MT7927 hardware.

## The Review Process: Five Versions

[Luiz Augusto von Dentz](https://lore.kernel.org/linux-bluetooth/?q=f%3Aluiz.dentz), the Bluetooth subsystem maintainer, reviewed each revision within hours. The series went through five versions over four weeks:

**v1 (2026-03-05):** Two patches. Luiz asked for per-device USB ID commits with `Tested-by` trailers, `lsusb -v` output confirming real hardware, and dmesg before/after. He also asked about the `Assisted-by: Claude Code` trailer - the kernel's [coding-assistants policy](https://docs.kernel.org/process/coding-assistants.html) requires disclosure when AI tools assist development.

**v2 (2026-03-25):** Split USB IDs into per-device commits. Added the ISO interface fix for single-alt-setting devices (13d3:3588). Collected `Tested-by` trailers from 8 community members. Dropped the `BTMTK_FIRMWARE_LOADED` skip logic per Sean Wang's feedback.

**v3 (2026-03-26):** Scoped the CHIPID workaround to VID/PID matching (not all zero-reads). Moved firmware to `mediatek/mt7927/` directory per Sean Wang. Luiz ran [sashiko](https://sashiko.dev/) - an automated AI review tool - which flagged two issues: an SDIO slab-out-of-bounds risk in `btmtk_setup_firmware_79xx` (used `hci_get_priv` with wrong struct) and a subsys reset failure (post-reset CHIPID validation always fails on MT6639).

**v4 (2026-03-30):** Fixed both sashiko findings. Passed `dev_id` as a parameter instead of using `hci_get_priv`. Skipped post-reset CHIPID validation for 0x6639.

**v5 (2026-03-31):** Fixed a cross-patch commit message coherence issue that Luiz caught - patch 8/8 described a 19-second initialization delay that patch 2/8 already fixes. Luiz applied v4 with the misleading note removed entirely, which was the simpler resolution.

### Merged

On 2026-03-31, Luiz pushed all 8 patches to [bluetooth-next](https://git.kernel.org/pub/scm/linux/kernel/git/bluetooth/bluetooth-next.git). The code will flow into mainline during the next merge window, shipping in Linux 7.1 or 7.2.

The MediaTek sign-off that initially blocked the series turned out to be unnecessary - Luiz accepted the patches based on the community testing evidence and code quality alone.

## The Community

The upstream process took four weeks. During that time, 8 community members across 6 countries provided hardware testing, bug reports, and `Tested-by` trailers. The [DKMS package](https://github.com/jetm/mediatek-mt7927-dkms) served as the bridge - giving users working Bluetooth while the upstream patches went through review. Community-maintained ports appeared for multiple distributions:

| Distribution | Maintainer | Repository |
|---|---|---|
| Arch Linux (AUR) | Javier Tia | [mediatek-mt7927-dkms](https://aur.archlinux.org/packages/mediatek-mt7927-dkms) |
| Ubuntu/Debian | giosal | [giosal/mediatek-mt7927-dkms](https://github.com/giosal/mediatek-mt7927-dkms) |
| NixOS (flake) | cmspam | [cmspam/mt7927-nixos](https://github.com/cmspam/mt7927-nixos) |
| NixOS (module) | clemenscodes | [clemenscodes/linux-mt7927](https://github.com/clemenscodes/linux-mt7927) |
| Bazzite (Fedora Atomic) | samutoljamo | [samutoljamo/bazzite-mt7927](https://github.com/samutoljamo/bazzite-mt7927) |

When the BT patches ship in mainline Linux 7.1 or 7.2, the Bluetooth portion of these packages becomes unnecessary - which is the goal.

## What's Left

1. ~~**BT driver patches**~~ - merged into bluetooth-next (2026-03-31)
2. **linux-firmware MR !946** - BT firmware blob, waiting for maintainer review
3. **WiFi driver patches** - 9-patch series on linux-wireless@, v4 under review
4. **Kernel release** - BT patches will ship in Linux 7.1 or 7.2

The [DKMS package](https://aur.archlinux.org/packages/mediatek-mt7927-dkms) remains necessary for WiFi and for users on kernels older than 7.1. WiFi support - a larger story involving 320 MHz EHT channels, per-chip IRQ maps, and ASPM quirks - will be covered separately.

## References

- [BT driver patches (merged, v4)](https://lore.kernel.org/linux-bluetooth/20260330-mt7927-bt-support-v4-0-cecc025e7062@jetm.me/) - linux-bluetooth mailing list
- [BT driver patches (v1, original submission)](https://lore.kernel.org/linux-bluetooth/177272816248.352280.12453518046823439297@jetm.me/) - linux-bluetooth mailing list
- [BT firmware MR !946](https://gitlab.com/kernel-firmware/linux-firmware/-/merge_requests/946) - linux-firmware GitLab
- [bluetooth-next tree](https://git.kernel.org/pub/scm/linux/kernel/git/bluetooth/bluetooth-next.git) - Luiz's bluetooth-next repository
- [mediatek-mt7927-dkms](https://github.com/jetm/mediatek-mt7927-dkms) - DKMS package source and documentation
- [OpenWRT mt76 issue #927](https://github.com/openwrt/mt76/issues/927) - MT7927 Linux support tracking
- [Kernel coding-assistants policy](https://docs.kernel.org/process/coding-assistants.html) - AI disclosure requirements
- [Phoronix: MediaTek MT7927 WiFi 7 Linux support](https://www.phoronix.com/news/MediaTek-MT7927-WiFi-Linux) - Phoronix coverage
- [Part 1: Enabling MT7927 Bluetooth]({{< ref "/posts/enabling-mt7927-bluetooth-on-linux" >}})
- [Part 2: WiFi - Wrong Driver, Wrong Chip]({{< ref "/posts/mt7927-wifi-the-missing-piece" >}})
