---
title: "MT7927 Bluetooth: From DKMS to Upstream"
date: 2026-03-05T10:00:00Z
draft: false
description: "Submitting MT7927 Bluetooth support to the Linux kernel - BT driver patches to linux-bluetooth, firmware to linux-firmware, and what the maintainer review process looks like from a first-time contributor's perspective."
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

This post covers what happened next: submitting all three layers upstream.

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

The BT driver changes are two patches:

1. **btmtk**: Add MT6639 (MT7927) hardware variant support - firmware naming, section filtering, initialization sequence
2. **btusb**: Add USB device IDs for six confirmed MT7927 hardware variants

### The Section Filter

The section filtering in patch 1 is the critical piece. The MT6639 firmware contains 9 sections, but only 5 are Bluetooth. The remaining sections are WiFi/other subsystem data. Without filtering by `dlmodecrctype & 0xFF == 0x01`, the driver sends non-Bluetooth sections to the chip, causing an irreversible subsystem hang. This matches the behavior of MediaTek's Windows driver.

A v2.1-20 bug fix caught a critical regression in this filter: the `dlmodecrctype` check wasn't gated on `dev_id == 0x6639`, meaning it would apply to all MediaTek Bluetooth chips - potentially breaking MT7921, MT7925, and every other chip that goes through the same firmware loading path. The upstream patch correctly scopes the filter to MT6639 only.

### Confirmed Hardware

Patch 2 adds USB IDs confirmed by community testing across multiple hardware platforms:

| USB ID | Hardware |
|--------|----------|
| `0489:e10f` | Gigabyte Z790 AORUS MASTER X |
| `0489:e116` | TP-Link Archer TBE550E |
| `0489:e13a` | ASUS ROG Crosshair X870E Hero |
| `0489:e0fa` | Lenovo Legion Pro 7 16ARX9 |
| `13d3:3588` | ASUS X870E-E / ASUS ProArt X870E-Creator WiFi |
| `0e8d:6639` | Generic MediaTek MT6639 |

New IDs can be added incrementally as users report them. The current set covers all known hardware.

## Luiz Dentz's Review

[Luiz Augusto von Dentz](https://lore.kernel.org/linux-bluetooth/?q=f%3Aluiz.dentz), the Bluetooth subsystem maintainer, reviewed both patches within hours of submission. His feedback was specific and practical.

**On the btmtk patch:** He asked for dmesg output showing the firmware loading before and after the patch, and noted that changes to MediaTek's vendor-specific code typically require a Signed-off-by from a MediaTek engineer. This is standard practice in the kernel - the vendor's sign-off confirms the changes match the hardware's actual behavior and won't break other MediaTek chips.

**On the btusb IDs:** He wanted `lsusb -v` output confirming the USB IDs are real devices, not guesses. Device ID patches occasionally get submitted based on documentation rather than hardware testing, so maintainers want proof.

I replied with:

- Full `lsusb -v` output from my ASUS ROG Crosshair X870E Hero (`0489:e13a`)
- dmesg before (firmware load failure) and after (successful initialization with `Device setup in 2602559 usecs`)
- Links to the [DKMS repository](https://github.com/jetm/mediatek-mt7927-dkms) and [OpenWRT tracking issue](https://github.com/openwrt/mt76/issues/927) showing community-wide hardware confirmation
- CC'd MediaTek engineers requesting their Signed-off-by on the btmtk changes

### The AI Disclosure

Luiz also asked about the `Assisted-by: Claude Code` trailer in the patches. The kernel's [coding-assistants policy](https://docs.kernel.org/process/coding-assistants.html) requires disclosure when AI tools are used during development. I clarified that the patches are human-authored - the AI assisted with code review, identifying the `sta_rec_mld` out-of-bounds bug, and catching the section filter scoping issue, but didn't generate the driver code itself.

This is a relatively new area for kernel development. The policy exists because the kernel community wants transparency about how code is produced, not because AI-assisted code is inherently problematic.

## Waiting on MediaTek

The BT driver patches are currently blocked on a MediaTek engineer providing their Signed-off-by on the btmtk changes. Chris Lu and Deren Wu from MediaTek have been active on linux-bluetooth recently - their [btmtk firmware retry patches](https://lkml.org/lkml/2026/2/3/338) were merged in February - so the request isn't going into a void.

Once the MediaTek sign-off arrives, Luiz can merge the patches into bluetooth-next. From there, they'll flow into Linus's tree during the next merge window and eventually reach stable kernels.

## The Community While We Wait

While the upstream process moves at kernel pace, the DKMS package continues to serve users today. Community-maintained ports have appeared for distributions beyond Arch Linux:

| Distribution | Maintainer | Repository |
|---|---|---|
| Arch Linux (AUR) | Javier Tia | [mediatek-mt7927-dkms](https://aur.archlinux.org/packages/mediatek-mt7927-dkms) |
| Ubuntu/Debian | giosal | [giosal/mediatek-mt7927-dkms](https://github.com/giosal/mediatek-mt7927-dkms) |
| NixOS (flake) | cmspam | [cmspam/mt7927-nixos](https://github.com/cmspam/mt7927-nixos) |
| NixOS (module) | clemenscodes | [clemenscodes/linux-mt7927](https://github.com/clemenscodes/linux-mt7927) |
| Bazzite (Fedora Atomic) | samutoljamo | [samutoljamo/bazzite-mt7927](https://github.com/samutoljamo/bazzite-mt7927) |

These packages carry the same patches but handle distro-specific differences: initramfs rebuilding, module signing, firmware paths, and package management. When the patches land upstream, these packages become unnecessary - which is the goal.

## What's Left

1. **MediaTek sign-off** on the btmtk patch - the single blocking item
2. **linux-firmware MR !946** merge - ready, waiting for maintainer review
3. **Kernel release cycle** - even after acceptance, patches flow through bluetooth-next to a stable release, typically 2-3 kernel cycles

Until then, the [DKMS package](https://aur.archlinux.org/packages/mediatek-mt7927-dkms) remains the way to get MT7927 Bluetooth working on Linux. WiFi support - which is a much larger story involving reverse-engineering, DMA ring layouts, and community collaboration - will be covered separately.

## References

- [BT firmware MR !946](https://gitlab.com/kernel-firmware/linux-firmware/-/merge_requests/946) - linux-firmware GitLab
- [BT driver patches on lore](https://lore.kernel.org/linux-bluetooth/177272816248.352280.12453518046823439297@jetm.me/) - linux-bluetooth mailing list
- [mediatek-mt7927-dkms](https://github.com/jetm/mediatek-mt7927-dkms) - DKMS package source and documentation
- [OpenWRT mt76 issue #927](https://github.com/openwrt/mt76/issues/927) - MT7927 Linux support tracking
- [Kernel coding-assistants policy](https://docs.kernel.org/process/coding-assistants.html) - AI disclosure requirements
- [Part 1: Enabling MT7927 Bluetooth]({{< ref "/posts/enabling-mt7927-bluetooth-on-linux" >}})
- [Part 2: WiFi - Wrong Driver, Wrong Chip]({{< ref "/posts/mt7927-wifi-the-missing-piece" >}})
