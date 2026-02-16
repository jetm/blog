---
title: "Enabling MediaTek MT7927 Bluetooth on Linux: A 15-Month Journey"
date: 2026-02-16T10:00:00Z
draft: false
description: "A technical deep-dive into enabling MT7927 Bluetooth support on Linux through kernel patches, firmware extraction, and DKMS packaging — from initial investigation to working AUR package."
ShowToc: true
ShowReadingTime: true
tags:
  - "linux"
  - "bluetooth"
  - "kernel"
  - "mediatek"
  - "mt7927"
  - "dkms"
  - "arch-linux"
  - "hardware"
categories:
  - "linux"
  - "hardware"
outputs:
  - "HTML"
---

In November 2024, I installed an ASUS ROG Crosshair X870E Hero motherboard — AMD's flagship X870E platform with a MediaTek MT7927 (Filogic 380) WiFi 7 chip. Running CachyOS, WiFi worked out of the box through the `mt7925e` driver. Bluetooth did not. What followed was a 15-month journey through kernel source, mailing lists, and community reverse-engineering that ended with a working Bluetooth 5.4 adapter and a published AUR package.

This is the story of how it got there.

## Act 1: The Three Missing Layers (November 2024)

Running `bluetoothctl show` returned nothing. No adapter, no controller, no error. Just silence. The `rfkill` output showed an `hci0` device, soft-blocked, but nothing behind it. Something was clearly half-working.

The first clue came from the kernel log:

```text
Bluetooth: hci0: Opcode 0x0c03 failed: -16
```

Opcode `0x0c03` is `HCI_Reset` — the most basic Bluetooth command. Error `-16` is `EBUSY`. The kernel's generic `btusb` driver had bound to the device by USB class code (E0/01/01, Bluetooth HCI), but it was sending a raw HCI reset instead of the vendor-specific initialization that MediaTek chips require.

The USB device ID told the story: `0489:e13a` (Foxconn / Hon Hai). This ID was missing from `btusb`'s quirks table — the lookup table that maps USB IDs to vendor-specific driver flags. Without it, `btusb` treated the MT7927 as a generic Bluetooth adapter.

Linux Bluetooth for MediaTek is a two-module stack: `btusb` handles USB transport, while `btmtk` handles vendor-specific firmware loading and chip initialization. Without the correct USB ID in `btusb`'s quirks table, `btmtk` never gets involved.

I wrote a DKMS patch to add the device ID with the `BTUSB_MEDIATEK` flag. After loading the patched module, a new error appeared:

```text
Bluetooth: hci0: Unsupported hardware variant (00006639)
```

Progress. The `btusb` driver now correctly identified the chip as MediaTek and handed initialization to `btmtk`, the MediaTek Bluetooth subsystem. But `btmtk` didn't recognize the hardware variant `0x6639`.

The MT7927 is marketed as a WiFi 7 chip, but its Bluetooth component is internally the MT6639 — a mobile SoC component. The variant ID `0x6639` had no case in `btmtk`'s switch statements, no firmware naming rule, and no reset procedure.

And even if the driver recognized the variant, there was no firmware. The `linux-firmware` repository had no MT6639 directory, no files, no pending merge requests. The chip needed `BT_RAM_CODE_MT6639_2_1_hdr.bin`, which didn't exist in any public Linux repository.

Three layers, all missing:

| Layer | Component | What's Missing |
|-------|-----------|----------------|
| 1 | btusb | USB ID `0489:e13a` not in quirks table |
| 2 | btmtk | HW variant `0x6639` unsupported |
| 3 | linux-firmware | No MT6639 BT firmware |

I documented everything in a `FINDINGS.md` and settled in to wait.

## Act 2: The Wait (December 2024 – January 2026)

The [OpenWRT mt76 issue #927](https://github.com/openwrt/mt76/issues/927) had been tracking MT7927 support since October 2024. The thread was a slow accumulation of users with the same problem: "my WiFi 7 card doesn't work on Linux," followed by confirmation that the chip was unsupported.

I monitored the Bluetooth LKML mailing list and the linux-firmware GitLab. Months passed. MediaTek firmware updates landed for MT7921, MT7925 — but nothing for MT6639.

Then in September 2025, a breakthrough appeared in the OpenWRT thread. A contributor named marcin-fm reverse-engineered the firmware format from an ASUS Windows driver package. The critical discovery: the MT7927's Bluetooth firmware was embedded inside `mtkwlan.dat` — a WiFi firmware container — not distributed as a separate Bluetooth file. The chip identifier wasn't MT7927 but MT6639, confirming the mobile SoC lineage.

By early February 2026, the frustration was widespread. Over at [CachyOS issue #688](https://github.com/CachyOS/linux-cachyos/issues/688), the response was blunt: "MT7927 is not supported on Linux. Replace it with an Intel wireless card." But upstream activity was finally accelerating. On February 3, Chris Lu from MediaTek posted a [patch series](https://lkml.org/lkml/2026/2/3/338) improving `btmtk`'s firmware retry flow and adding reset mechanisms for failed downloads. These patches were [merged into bluetooth-next](https://lkml.org/lkml/2026/2/11/1085) on February 11.

On February 8, Jean-François Marlière posted a [comprehensive patch](https://lkml.org/lkml/2026/2/8/410) fixing MT7927/MT6639 firmware loading. He identified three specific issues: missing switch cases for variant `0x6639`, incorrect firmware naming (`_1_1_hdr.bin` instead of `_2_1_hdr.bin`), and — critically — a section filtering problem. The MT6639 firmware contains 9 sections, but only 5 are Bluetooth. The remaining sections are WiFi/other subsystem data. Without filtering, the driver would send non-Bluetooth sections to the chip, causing an irreversible subsystem hang. Marlière's patch matched the Windows driver behavior: only download sections where `dlmodecrctype & 0xFF == 0x01`.

The pieces were coming together, but none of this had reached a stable kernel release.

## Act 3: The Build (February 2026)

On February 14, a developer named [clemenscodes](https://github.com/clemenscodes/linux-mediatek-mt6639-bluetooth-kernel-module) published a GitHub repository with a unified patch for kernel 6.19 that addressed all three layers: USB ID, HW variant support, and firmware loading with section filtering. Two days later, [kernel bugzilla #221096](https://bugzilla.kernel.org/show_bug.cgi?id=221096) was filed referencing the patch, with confirmation of successful testing on multiple devices.

I decided to package everything into a DKMS-based AUR package. The challenges were immediate.

**Firmware extraction.** The BT firmware isn't publicly available. It's embedded inside `mtkwlan.dat` within ASUS's Windows WiFi driver ZIP. An `extract_firmware.py` script (from the clemenscodes repo, based on marcin-fm's earlier work) parses the binary container, locates the `BT_RAM_CODE_MT6639_2_1_hdr.bin` string, and extracts the 688 KB firmware blob.

**Dual-module build.** The original DKMS package only built `btusb.ko`. But the clemenscodes patch modifies both `btusb.c` and `btmtk.c` — and `btmtk` compiles to a separate kernel module, `btmtk.ko`. The DKMS configuration needed a `PRE_BUILD` script that downloads both source files from kernel.org, applies the patch, and generates a Makefile building both modules.

**The initialization puzzle.** After installing the package and loading the modules, the firmware loaded and the setup sequence completed:

```text
Bluetooth: hci0: HW/SW Version: 0x00000000, Build Time: 20250606201235
Bluetooth: hci0: Device setup in 2602559 usecs
```

But `bluetoothctl show` returned nothing. The kernel log showed `setting interface failed (22)` — a non-fatal error about the SCO/ISO USB alternate interface — followed by a successful setup completion. The kernel had created `hci0` in sysfs, rfkill showed it unblocked, and BlueZ's D-Bus service had registered A2DP endpoints — yet the management layer couldn't see the adapter.

The root cause was timing: the stock `btusb` module had loaded at boot and sent incorrect commands to the chip, corrupting its state. A USB device reset — deauthorizing and reauthorizing the USB port via sysfs while reloading the modules — cleared the stuck state:

```text
Controller 78:46:5C:9B:C2:C2 (public)
    Name: jetm-rog-crosshair-hero-x870e
    Manufacturer: 0x0046 (MediaTek)
    Powered: yes
    Roles: central, peripheral
```

A fully functional Bluetooth 5.4 adapter, 15 months after the first investigation.

To confirm it worked beyond just adapter visibility, I paired an EarFun Air Pro 4. The `bluetoothctl` session showed A2DP media endpoints registering and an audio transport coming up:

```text
[NEW] Media /org/bluez/hci0
    SupportedUUIDs: 0000110a-0000-1000-8000-00805f9b34fb
    SupportedUUIDs: 0000110b-0000-1000-8000-00805f9b34fb
[NEW] Endpoint /org/bluez/hci0/dev_70_5A_6F_6B_4E_E7/sep5
[NEW] Endpoint /org/bluez/hci0/dev_70_5A_6F_6B_4E_E7/sep6
[NEW] Endpoint /org/bluez/hci0/dev_70_5A_6F_6B_4E_E7/sep8
[NEW] Endpoint /org/bluez/hci0/dev_70_5A_6F_6B_4E_E7/sep10
[NEW] Transport /org/bluez/hci0/dev_70_5A_6F_6B_4E_E7/sep8/fd0
Agent registered
[EarFun Air Pro 4]> devices
Device 70:5A:6F:6B:4E:E7 EarFun Air Pro 4
```

Audio streaming worked immediately — no additional configuration required.

The AUR package [`btusb-mt7927-dkms`](https://aur.archlinux.org/packages/btusb-mt7927-dkms) installs the DKMS source tree and firmware. On subsequent boots, the patched modules load before the stock ones, so the initialization corruption doesn't recur.

## Looking Ahead

As of February 2026, none of the three layers have reached mainline Linux. Marlière's LKML patch is pending review, the firmware hasn't been submitted to linux-firmware, and the USB device ID is missing from upstream btusb. Until all three land in a stable kernel release, out-of-tree solutions like this DKMS package remain the only path.

But the path exists. For anyone with MT7927 hardware waiting for Linux Bluetooth support: the pieces are here, the community has assembled them, and they work.

## References

- [Kernel Bugzilla #221096](https://bugzilla.kernel.org/show_bug.cgi?id=221096) — MT7927 Bluetooth support request
- [OpenWRT mt76 issue #927](https://github.com/openwrt/mt76/issues/927) — MT7927 Linux support tracking (Oct 2024–)
- [clemenscodes kernel module repo](https://github.com/clemenscodes/linux-mediatek-mt6639-bluetooth-kernel-module) — Unified 6.19 patch and firmware extraction
- [CachyOS issue #688](https://github.com/CachyOS/linux-cachyos/issues/688) — "MT7927 is not supported on Linux"
- [Marlière's LKML patch](https://lkml.org/lkml/2026/2/8/410) — btmtk fix for MT7927/MT6639 firmware loading
- [Chris Lu's MediaTek patches](https://lkml.org/lkml/2026/2/3/338) — btmtk firmware retry and reset improvements
- [linux-firmware GitLab](https://gitlab.com/kernel-firmware/linux-firmware) — Upstream firmware repository (no MT6639 as of Feb 2026)
- [btusb-mt7927-dkms AUR package](https://aur.archlinux.org/packages/btusb-mt7927-dkms) — DKMS package for Arch Linux
