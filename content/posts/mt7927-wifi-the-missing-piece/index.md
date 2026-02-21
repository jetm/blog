---
title: "MT7927 WiFi on Linux: Wrong Driver, Wrong Chip, No Driver"
date: 2026-02-20T12:00:00Z
draft: true
description: "I spent a day applying the wrong kernel patches to the wrong driver for the wrong chip before figuring out that MT7927 WiFi doesn't actually work on Linux yet. Here's how MediaTek's naming burned me and what I built anyway."
ShowToc: true
ShowReadingTime: true
tags:
  - "linux"
  - "wifi"
  - "kernel"
  - "mediatek"
  - "mt7927"
  - "mt7925"
  - "mt7902"
  - "dkms"
  - "arch-linux"
  - "hardware"
  - "mt76"
categories:
  - "linux"
  - "hardware"
outputs:
  - "HTML"
---

In my [previous post](/posts/enabling-mt7927-bluetooth-on-linux/), I documented the 15-month journey to get Bluetooth working on the MediaTek MT7927. The `btusb-mt7927-dkms` AUR package patched three missing layers — USB device ID, hardware variant support, and firmware extraction — to bring up a fully functional Bluetooth 5.4 adapter.

WiFi was next. I spent most of a day on it and came out the other side with the wrong driver applied, a hard crash course in MediaTek's chip numbering, and the realization that MT7927 WiFi straight up does not work on Linux. Not with patches, not with hacks, not yet.

## MT7902 Patches Show Up on lore

On February 19, Sean Wang from MediaTek posted an [11-patch series](https://lore.kernel.org/linux-wireless/20260219004007.19733-1-sean.wang@kernel.org/) to the linux-wireless mailing list adding MT7902 WiFi 6E support to the `mt7921e` driver. New PCI device ID, IRQ mappings, firmware paths, MCU commands — the works.

MT7902. MT7927. Both MediaTek, both recent, both wireless. I figured MT7902 was probably the WiFi silicon inside MT7927, same way MT6639 turned out to be the Bluetooth silicon. Worth a shot.

## Applying the Patches

The mt76 driver lives at `drivers/net/wireless/mediatek/mt76/` in the kernel tree. I pulled about 60 source files for kernel 6.19.3 — the mt76 core, mt792x shared library, mt7921 subdirectory — and grabbed the patch series with `b4 am`:

```text
$ b4 am 20260219004007.19733-1-sean.wang@kernel.org
Analyzing 11 messages in the thread
---
  [PATCH 01/11] wifi: mt76: mt792x: rename is_mt7921 to is_connac2
  ...
  [PATCH 11/11] wifi: mt76: mt7921: enable MT7902 WiFi 6E support
---
```

`b4` instead of `curl` — lore.kernel.org 403s direct HTTP fetches from non-browser user agents.

Patches didn't apply cleanly. Patch 3 failed on `mt7921/pci.c` — the context had shifted between the patch's base and v6.19.3 (`regs` became `pcim_iomap_table(pdev)[0]`). Patch 1 renamed `is_mt7921()` to `is_connac2()` in the header but missed a call at `mt792x_core.c:694`. Fixed both by hand.

After that, I wrote Kbuild files for out-of-tree compilation, built it all as DKMS modules. Seven modules: `mt76`, `mt76-connac-lib`, `mt792x-lib`, `mt7921-common`, `mt7921e`, plus the existing `btusb` and `btmtk`.

CachyOS uses Clang for its kernels, so the DKMS config auto-detects the toolchain:

```bash
_LLVM="$(grep -qs '^CONFIG_CC_IS_CLANG=y' ${kernel_source_dir}/.config && echo LLVM=1)"
```

All compiled, all installed. Loaded the modules:

```text
$ lspci -nnk -s 0b:00.0
0b:00.0 Network controller [0280]: MEDIATEK Corp. MT7927 ... [14c3:7927]
    Kernel modules: mt7921e
```

Nothing bound. `mt7921e` was loaded, but the PCI subsystem wouldn't match it. The MT7902 patches add PCI ID `0x7902`. My hardware is `0x7927`. Not the same chip.

## MediaTek's Numbering Makes No Sense

This is where I wasted the most time. MediaTek's chip numbers look sequential but they're not. Here's the actual map:

| Chip | Type | Generation | Driver | PCI ID |
|------|------|-----------|--------|--------|
| MT7921 | WiFi 6E | Wi-Fi 6E | mt7921e | 14c3:7961 |
| MT7902 | WiFi 6E | Wi-Fi 6E (new variant) | mt7921e | 14c3:7902 |
| MT7925 | WiFi 7 | Wi-Fi 7 | mt7925e | 14c3:7925 |
| MT7927 | WiFi 7 + BT combo | Wi-Fi 7 | mt7925e | 14c3:7927 |
| MT6639 | BT (inside MT7927) | BT 5.4 | btusb/btmtk | 0489:e13a |

MT7927 is not "newer than MT7925." It *is* MT7925, bundled with a Bluetooth controller (MT6639) in one combo package. MediaTek calls it Filogic 380. And MT7902? Completely different product line — WiFi 6E, older generation, uses the mt7921 driver. The number is between 7921 and 7925 but it has nothing to do with either of them architecturally.

```
MT7927 (combo module on the motherboard)
├── BT:   MT6639 → btusb/btmtk driver (USB, 0489:e13a)
└── WiFi: MT7925-architecture → mt7925e driver (PCIe, 14c3:7927)

MT7902 (separate chip, unrelated to MT7927)
└── WiFi: MT7921-architecture → mt7921e driver (PCIe, 14c3:7902)
```

So I'd been patching mt7921e all afternoon. Wrong driver entirely.

## Trying mt7925e

I pulled the mt7925 source (14 files), wrote a one-line patch adding `0x7927` to the PCI device table:

```c
static const struct pci_device_id mt7925_pci_device_table[] = {
    { PCI_DEVICE(PCI_VENDOR_ID_MEDIATEK, 0x7925),
        .driver_data = (kernel_ulong_t)MT7925_FIRMWARE_WM },
    { PCI_DEVICE(PCI_VENDOR_ID_MEDIATEK, 0x0717),
        .driver_data = (kernel_ulong_t)MT7925_FIRMWARE_WM },
+   { PCI_DEVICE(PCI_VENDOR_ID_MEDIATEK, 0x7927),
+       .driver_data = (kernel_ulong_t)MT7925_FIRMWARE_WM },
    { },
};
```

Rebuilt. Loaded. It bound:

```text
$ lspci -nnk -s 0b:00.0
0b:00.0 Network controller [0280]: MEDIATEK Corp. MT7927 ... [14c3:7927]
    Kernel driver in use: mt7925e
```

Then:

```text
mt7925e 0000:0b:00.0: ASIC revision: 0000
mt7925e 0000:0b:00.0: Message 00000010 (seq 2) timeout
mt7925e 0000:0b:00.0: Failed to get patch semaphore
...
mt7925e 0000:0b:00.0: hardware init failed
```

`ASIC revision: 0000` means the driver is reading all zeros from hardware registers. PCIe BARs are mapped fine (`Region 0: Memory at f2000000`), but the chip isn't responding. Everything after that — the semaphore timeouts, the MCU failures — is just the driver talking to a wall.

I tried disabling ASPM, enabling Bus Master manually with `setpci`, doing a full PCI remove/rescan cycle. Nothing. The MT7927 WiFi silicon needs a different initialization path than MT7925 — something around DMA firmware transfer — and that code doesn't exist in mainline mt76.

## Nobody Has This Working

I went looking for anyone who'd gotten further. The best I found was [ehausig/mt7927](https://github.com/ehausig/mt7927) on GitHub — a community reverse-engineering project. They got past PCI binding: firmware loads into kernel memory, register access works through BAR2 control registers, no crashes. But the firmware gets stuck at state `0xffff10f1` — "waiting for DMA firmware transfer." The DMA descriptor setup, buffer headers, MCU trigger sequence — nobody's cracked it. The project is on hold.

[OpenWRT mt76 issue #927](https://github.com/openwrt/mt76/issues/927) has been open since October 2024. Eighty-nine comments, forty-six upvotes, every reporter saying the same thing. No MediaTek developer has ever replied.

Same story on [Arch](https://bbs.archlinux.org/viewtopic.php?id=303402), [Manjaro](https://forum.manjaro.org/t/mediatek-wi-fi-7-mt7927/154995), [Garuda](https://forum.garudalinux.org/t/support-for-mediatek-mt7927/34043), [Fedora](https://discussion.fedoraproject.org/t/mt7927-bluetooth-support-in-kernal/147702), [Linux Mint](https://forums.linuxmint.com/viewtopic.php?t=433401), [MX Linux](https://forum.mxlinux.org/viewtopic.php?t=82139). The CachyOS team's response was [the most direct](https://github.com/CachyOS/linux-cachyos/issues/688): "MT7927 is not supported on Linux. Replace it with an Intel wireless card."

## The Package

I shipped it anyway. Bluetooth works, the WiFi infrastructure is in place, and the MT7902 modules are free to include.

The `btusb-mt7927-dkms` package became `mediatek-mt7927-dkms`. Nine DKMS modules:

| Module | Purpose | Status |
|--------|---------|--------|
| `btusb` | Bluetooth USB transport | Working |
| `btmtk` | MediaTek BT firmware loading | Working |
| `mt76` | Core mt76 framework | Working |
| `mt76-connac-lib` | Connac library (shared) | Working |
| `mt792x-lib` | 792x shared library | Working |
| `mt7921-common` | MT7921/MT7902 common | Working (for MT7902 HW) |
| `mt7921e` | MT7902 WiFi 6E PCIe | Working (for MT7902 HW) |
| `mt7925-common` | MT7925/MT7927 common | Builds, installs |
| `mt7925e` | MT7927 WiFi 7 PCIe | Binds, firmware init fails |

MT7902 modules are in there because the mt76 build already pulls the shared core (`mt76`, `mt76-connac-lib`, `mt792x-lib`). Adding `mt7921e` on top costs nothing — same dependency chain — and it gives MT7902 hardware users Sean Wang's WiFi 6E patches for free.

The PKGBUILD sources firmware through a priority chain: a pre-placed `BT_RAM_CODE_MT6639_2_1_hdr.bin` blob, a raw `mtkwlan.dat` from any OEM driver package, a MediaTek WiFi driver ZIP matching common naming patterns, or — as a fallback — auto-download from ASUS's CDN using code [contributed by Eadinator](https://github.com/openwrt/mt76/issues/927#issuecomment-3936022734) that authenticates against their TokenHQ API for a signed CloudFront URL. From there it extracts Bluetooth firmware, pulls mt76 source from kernel.org, applies three patches, and installs everything as a DKMS tree that rebuilds on kernel updates.

The same MT7927/MT6639 silicon ships in more than just ASUS boards. The patches now cover multiple OEM variants:

| Hardware | BT USB ID | WiFi PCI ID |
|----------|-----------|-------------|
| ASUS ROG Crosshair X870E Hero | `0489:e13a` | `14c3:7927` |
| Lenovo Legion Pro 7 16ARX9 | `0489:e0fa` | `14c3:7927` |
| Foxconn/Azurewave modules | — | `14c3:6639` |
| AMD RZ738 (MediaTek MT7927) | — | `14c3:0738` |

The firmware is the same across all of them — only the device IDs differ. Any OEM's MediaTek WiFi driver package containing `mtkwlan.dat` works as a firmware source, so users with Lenovo or Foxconn hardware don't need the ASUS ZIP.

When someone figures out the MT7927 DMA init — MediaTek pushing code upstream, the ehausig project resuming, whoever — users with this package just need an update. The mt7925e module is already there with the right PCI IDs for all known variants. The firmware files are on disk. Only the init code needs to change.

## Looking Ahead

If you're shopping for a motherboard and care about Linux WiFi, check whether the wireless chip has a mainline driver before buying. `14c3:7927`, `14c3:6639`, and `14c3:0738` do not, as of February 2026. Intel AX-series cards remain the safe bet.

If you already have MT7927 hardware — whether it's an ASUS board, Lenovo laptop, or AMD RZ738 module — Bluetooth works today via the [mediatek-mt7927-dkms](https://aur.archlinux.org/packages/mediatek-mt7927-dkms) AUR package. WiFi doesn't. The package has the mt7925e module ready with PCI IDs for all known variants, so when the upstream fix lands, it's a package update away.

And if you're trying to figure out which MediaTek chip you actually have — ignore the model number. Run `lspci -nn`, grab the PCI ID, and search for that instead. It'll save you an afternoon.

## References

- [Sean Wang's MT7902 patch series](https://lore.kernel.org/linux-wireless/20260219004007.19733-1-sean.wang@kernel.org/) — 11 patches adding MT7902 WiFi 6E to mt7921e
- [ehausig/mt7927](https://github.com/ehausig/mt7927) — Community driver research project (on hold)
- [OpenWRT mt76 issue #927](https://github.com/openwrt/mt76/issues/927) — MT7927 Linux support tracking (Oct 2024–)
- [OpenWRT mt76 issue #1022](https://github.com/openwrt/mt76/issues/1022) — MT7925e ASIC revision 0000 analysis
- [clemenscodes kernel module repo](https://github.com/clemenscodes/linux-mediatek-mt6639-bluetooth-kernel-module) — MT6639 Bluetooth patch and firmware extraction
- [CachyOS issue #688](https://github.com/CachyOS/linux-cachyos/issues/688) — "Replace it with an Intel wireless card"
- [mediatek-mt7927-dkms AUR package](https://aur.archlinux.org/packages/mediatek-mt7927-dkms) — DKMS package for Arch Linux
- [Previous post: Enabling MT7927 Bluetooth on Linux](/posts/enabling-mt7927-bluetooth-on-linux/)
