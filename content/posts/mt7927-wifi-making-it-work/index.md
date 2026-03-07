---
title: "MT7927 WiFi on Linux: Making It Work"
date: 2026-03-06T18:00:00Z
draft: false
description: "How I got MediaTek MT7927 WiFi 7 working on Linux by reverse-engineering the MT7927 initialization, DMA ring layout, and firmware quirks - extending the existing mt7925e kernel driver with 18 patches. Now community-tested on 10+ hardware platforms across 2.4/5/6GHz with 320MHz (2+ Gbps), suspend/resume, and ASPM fixes all working. Series submitted to linux-wireless@."
ShowToc: true
ShowReadingTime: true
tags:
  - "linux"
  - "wifi"
  - "kernel"
  - "mediatek"
  - "mt7927"
  - "mt7925"
  - "upstream"
  - "dkms"
  - "arch-linux"
  - "hardware"
  - "mt76"
  - "reverse-engineering"
categories:
  - "linux"
  - "hardware"
outputs:
  - "HTML"
---

In my [previous post]({{< ref "/posts/mt7927-wifi-the-missing-piece" >}}), I ended with a wall:

```text
mt7925e 0000:0b:00.0: ASIC revision: 0000
mt7925e 0000:0b:00.0: Message 00000010 (seq 2) timeout
mt7925e 0000:0b:00.0: Failed to get patch semaphore
mt7925e 0000:0b:00.0: hardware init failed
```

The mt7925e driver bound to the MT7927's WiFi hardware, but registers returned zeros. The chip sat behind PCIe doing nothing. The [ehausig/mt7927](https://github.com/ehausig/mt7927) project had gotten firmware into kernel memory but stalled at DMA state `0xffff10f1` - "waiting for firmware transfer." Nobody had gotten past it.

This post is the story of getting past it.

## The Breakthrough

On February 21, I found [Loong0x00/mt7927](https://github.com/Loong0x00/mt7927) on GitHub - a standalone Linux driver for the MT7927 WiFi chip, reverse-engineered from the Windows driver. Where ehausig had mapped registers and documented state machines, Loong0x00 had built a working driver. Not a kernel module you could upstream - it was a standalone implementation with its own DMA engine - but it booted the chip, loaded firmware, created a `wlan0` interface, and completed WiFi scans.

The code confirmed what I'd suspected but couldn't prove: the MT7927's WiFi silicon is architecturally MT7925, but its bus-level initialization is completely different. The chip doesn't speak the MT7925 protocol until you wake it up through a different door.

I decided against using the standalone driver directly. It would need constant maintenance against kernel API changes, couldn't leverage the existing mt76 framework's power management, roaming, or multi-band support, and would be impossible to propose for upstream. Instead, I'd use Loong0x00's code as a reference to understand what the MT7927 hardware actually needs, then implement those changes as patches to the existing mt7925e driver.

The initial change was ~370 lines across 14 commits. It's now grown to 18 patches with community contributions for 320MHz, ASPM, and stability fixes. Here's how it started and what each piece does.

## Phase 1: The Register Wall

The first problem is that `ASIC revision: 0000`. The mt7925e probe reads CHIPID and REV registers to identify the hardware:

```c
mdev->rev = (mt76_rr(dev, MT_HW_CHIPID) << 16) |
            (mt76_rr(dev, MT_HW_REV) & 0xff);
```

On MT7925, this returns `0x79250001` or similar. On MT7927, it returns `0x00000000`. Not because the registers don't exist - the chip is there, PCIe BARs are mapped, config space reads fine - but because the MT7927 combo chip has an additional bus fabric called CBInfra (ConnectaBus Infrastructure) that sits between PCIe and the WiFi subsystem. Before configuring the CBTOP (ConnectaBus Top) address remap, every MMIO read to WiFi registers returns zero.

From Loong0x00's init sequence, the CBTOP remap configuration is:

```c
/* Wake CONNINFRA - required before CBInfra register access */
mt76_wr(dev, MT_CBINFRA_WAKEPU_TOP, 0x1);
usleep_range(1000, 2000);

/* Configure CBTOP PCIe address remap for WF and BT */
mt76_wr(dev, MT_CBINFRA_MISC0_REMAP_WF, 0x74037001);
mt76_wr(dev, MT_CBINFRA_MISC0_REMAP_BT, 0x70007000);
```

These magic values tell CBInfra how to route PCIe MMIO accesses to the WiFi and Bluetooth subsystems. Without them, every register read gets swallowed by the bus fabric.

After CBTOP remap, CHIPID reads come back as `0x7927`. Progress. But the MT7925 init sequence still fails because MT7927 needs its own chip initialization rather than the standard `mt792x_wfsys_reset()` path. The CBInfra chip init sequence, also from the reference driver:

1. Enable EMI sleep protect
2. WF subsystem reset via CBInfra RGU (assert, wait 1ms, deassert, wait 5ms)
3. Take MCU ownership
4. Poll `ROMCODE_INDEX` for MCU idle value `0x1D1E`
5. Configure MCIF remap for host DMA
6. Disable PCIe sleep
7. Clear CONNINFRA wakeup

Step 4 is the gate. The MCU boots from ROM and signals readiness by writing `0x1D1E` to a status register. If that value never appears, the chip never initialized. On my hardware it appeared within ~200ms - the chip was alive.

After these changes, the probe logs showed:

```text
mt7925e 0000:0b:00.0: MT7927 CBTOP remap configured
mt7925e 0000:0b:00.0: ASIC revision: 79270000
mt7925e 0000:0b:00.0: MT7927 chip init complete, starting DMA
```

Registers read correctly. Chip ID detected. But firmware loading still timed out. The MCU was awake and ready to talk, but the DMA engine couldn't deliver messages to it.

## Phase 2: The DMA Ring Mismatch

PCIe WiFi chips use DMA rings - circular buffers in host memory - to exchange data and commands with the chip's MCU. The host writes a command descriptor to a TX ring and bumps the write pointer; the chip's DMA engine reads it and delivers the payload to the MCU. Responses come back on RX rings.

MT7925 uses this ring layout:

| Ring | Index | Purpose |
|------|-------|---------|
| TX 0 | 0 | Data TX |
| TX 15 | 15 | MCU WM commands |
| TX 16 | 16 | Firmware download |
| **RX 0** | **0** | **MCU event responses** |
| **RX 2** | **2** | **WiFi data frames** |

MT7927 uses different RX ring indices:

| Ring | Index | Purpose |
|------|-------|---------|
| TX 0 | 0 | Data TX (same) |
| TX 15 | 15 | MCU WM commands (same) |
| TX 16 | 16 | Firmware download (same) |
| **RX 6** | **6** | **MCU event responses** |
| **RX 4** | **4** | **WiFi data frames** |
| **RX 7** | **7** | **Management frames** |

TX rings are the same. RX rings are completely different. The mt7925e driver was sending MCU commands on TX ring 15 (correct) and waiting for responses on RX ring 0 (wrong). The chip was responding on RX ring 6, which nobody was listening to. Every MCU command timed out.

This also explained the `Failed to get patch semaphore` error. The firmware download handshake is: (1) send semaphore request on TX ring 16, (2) chip responds with grant on RX MCU ring. With the wrong RX ring index, the grant was delivered to ring 6, the driver polled ring 0, and the semaphore acquisition timed out.

The fix is a separate DMA init function - `mt7927_dma_init()` - that allocates rings at the correct indices and a corresponding IRQ map that watches the right interrupt bits:

```c
static const struct mt792x_irq_map mt7927_irq_map = {
    .host_irq_enable = MT_WFDMA0_HOST_INT_ENA,
    .tx = {
        .all_complete_mask = MT_INT_TX_DONE_ALL,
        .mcu_complete_mask = MT_INT_TX_DONE_MCU,
    },
    .rx = {
        .data_complete_mask = HOST_RX_DONE_INT_ENA4,
        .wm_complete_mask  = HOST_RX_DONE_INT_ENA6,
        .wm2_complete_mask = HOST_RX_DONE_INT_ENA7,
    },
};
```

MT7925 watches `HOST_RX_DONE_INT_ENA0` and `HOST_RX_DONE_INT_ENA2`. MT7927 watches bits 4, 6, and 7. Without the correct interrupt mask, the NAPI poll never triggers for incoming MCU events.

The DMA engine also uses different prefetch registers. MT7925 programs per-ring `EXT_CTRL` registers directly. MT7927 uses "packed prefetch" - four 32-bit configuration registers (`PREFETCH_CFG0` through `CFG3`) that encode ring base addresses and depths in a compact format, plus per-ring `EXT_CTRL` values. Values came directly from Loong0x00's driver.

With the correct DMA ring layout and prefetch config, MCU commands started succeeding. Firmware loaded. An interface was created. Then scanning timed out.

## The ROM Paradox

Before the DMA fix worked, I spent days on a maddening problem. DMA rings would initialize correctly - I could see the host write descriptors and bump CIDX (CPU index) - but DIDX (DMA index, the chip's read pointer) never advanced. The chip's DMA engine was ignoring the rings entirely.

The cause was the mt76 power management path. The standard mt7925e initialization does a `SET_OWN` / `CLR_OWN` cycle at two points: once early in probe, and once in `mt7925e_mcu_init()`. SET_OWN tells the chip "you own the hardware," CLR_OWN says "host is taking over."

On MT7925, this is harmless. On MT7927, `CLR_OWN` triggers the ROM bootloader to reinitialize the WFDMA (WiFi DMA) engine. Every register, every prefetch configuration, every ring base address - wiped back to ROM defaults. So the DMA init sequence would carefully configure everything, then `mt7925e_mcu_init()` would do a `SET_OWN` / `CLR_OWN` and obliterate the entire configuration.

The fix has three parts:

1. **Skip early CLR_OWN in probe.** The probe's CLR_OWN happens before CBTOP remap is configured, so the ROM's WFDMA init fails anyway (it can't access the registers it needs). Skip it for MT7927 and do the CLR_OWN later, inside `mt7927_dma_init()`, after CBTOP and CBInfra are ready.

2. **Skip CLR_OWN in `mt7925e_mcu_init()`.** After `mt7927_dma_init()` configures the DMA rings, `mt7925e_mcu_init()` must not do another SET_OWN/CLR_OWN cycle. For MT7927, the function skips the power management handshake entirely to preserve DMA state.

3. **Fix the PM recovery path.** The mt792x framework has a power management work queue (`mt792x_pm_wake_work`) that periodically does CLR_OWN to wake the chip. Each wake triggers `mt792x_dma_enable()`, which restores DMA state - but only for MT7925's register configuration. MT7927 needs three additional GLO_CFG operations after each wake: setting `ADDR_EXT_EN` (BIT(26)), setting `FW_DWLD_BYPASS_DMASHDL` (BIT(9)), and clearing `CSR_LBK_RX_Q_SEL_EN` (BIT(20)). Without these, the DMA engine won't advance descriptors after a PM cycle. The prefetch configuration and ring base addresses are already handled by existing code paths, and `GLO_CFG_EXT1` BIT(28) is handled by the `is_mt7925()` block in `mt792x_dma_enable()` (since `is_mt7925()` returns true for chip 0x7927). With the recovery path fixed, runtime PM is enabled.

With all three, DMA ring descriptors started advancing. MCU commands completed in under 10ms. Firmware loaded. The interface came up. `iw dev` showed a `wlan0`.

```text
$ iw dev
phy#0
    Interface wlan0
        type managed
```

Scanning still didn't work, but for a new reason.

## Phase 3: The Silent Authentication Failure

With DMA working and firmware loaded, `iw scan trigger` returned success and scanned networks appeared in `iw scan dump`. But `iw connect` and `wpa_supplicant` both failed - the driver would attempt to authenticate but never receive a response. No error in dmesg, no timeout message, just silence.

The issue was in the mac80211 operations table. When `mt7925e` loads firmware, it reads a "feature trailer" - metadata appended to the firmware binary that advertises chip capabilities. One critical capability flag is `MT792x_FW_CAP_CNM` (Channel Navigation Manager). When CNM is present, the driver uses its native channel context operations, which include `mgd_prepare_tx` - a callback that sends a Remain-on-Channel (ROC) command to the firmware before sending authentication frames. ROC tells the firmware "stay on this channel and listen for the response."

MT7927's firmware doesn't include the connac2 feature trailer. The firmware loads fine - it's valid code - but the trailer parsing returns zero for all capabilities. Without the CNM flag, `mt792x_get_mac80211_ops()` replaces the native chanctx operations with emulated stubs and sets `mgd_prepare_tx` to NULL. This means:

1. No ROC before authentication - the firmware doesn't know to stay on the target channel
2. Emulated channel context instead of hardware channel context - the firmware never receives a channel assignment for the BSS
3. Authentication frames are sent but the firmware has no channel context, so they're silently dropped

The fix: detect MT7927 hardware and force the CNM flag, then restore the original mt7925 operations table:

```c
if ((pdev->device == 0x6639 || pdev->device == 0x7927) &&
    !(features & MT792x_FW_CAP_CNM)) {
    features |= MT792x_FW_CAP_CNM;
    memcpy(ops, &mt7925_ops, sizeof(*ops));
}
```

With CNM forced, `wpa_supplicant` completed the full WPA2 handshake. Connected to my 2.4GHz network at 258 Mbit/s. First data over MT7927 WiFi on Linux.

5GHz didn't work.

## Phase 4: The Missing Band

Scanning on 5GHz returned nothing. Not "no networks found" - the scan itself showed zero activity on 5GHz channels. Survey data confirmed it: `channel active time: 0 ms` for every 5GHz channel. The driver was sending 5GHz scan requests to the firmware (I could see the MCU commands with band=2 and 28 channels), but the firmware was ignoring them.

The answer was DBDC - Dual-Band Dual-Concurrent mode. MT7925 firmware handles DBDC automatically. The driver has a function `mt7925_mcu_set_dbdc()` that sends a `MCU_UNI_CMD(SET_DBDC_PARMS)` command with `mbmc_en=1`, but it's never called in the MT7925 init path because the firmware enables it internally.

MT7927 firmware doesn't. Without an explicit DBDC enable, the firmware defaults to single-band mode - 2.4GHz only. Every 5GHz scan request is silently discarded.

One line in `__mt7925_init_hardware()`, after `mt7925_mac_init()`:

```c
if (is_mt7927(&dev->mt76)) {
    ret = mt7925_mcu_set_dbdc(&dev->mphy, true);
    if (ret)
        dev_warn(dev->mt76.dev, "MT7927 DBDC enable failed: %d\n", ret);
}
```

After this, 5GHz networks appeared. Connected to my 5GHz AP at 1200.9 Mbit/s - HE-MCS 11, 80MHz, 2x2 MIMO.

## Phase 5: The Probe Bug

After publishing v2.0 of the package, [Loong0x00](https://github.com/Loong0x00) found a critical bug: `is_mt7927_hw` was used uninitialized in `mt7925_pci_probe()`.

The variable was declared at the top of the function but only assigned 28 lines after its first use - at the IRQ map selection:

```c
bool is_mt7927_hw;  // uninitialized

// ... 28 lines later ...

if (is_mt7927_hw)           // reads stack garbage
    dev->irq_map = &mt7927_irq_map;  // MT7927 rings 4/6/7
else
    dev->irq_map = &irq_map;         // MT7925 rings 0/2

// ... 28 more lines later ...

is_mt7927_hw = (pdev->device == 0x6639 || pdev->device == 0x7927);  // too late
```

On x86_64, uninitialized stack variables are typically zero. So the condition was always false, selecting the MT7925 IRQ map (RX rings 0/2) instead of `mt7927_irq_map` (RX rings 4/6/7). The driver polled the wrong rings for MCU responses, every command timed out, and the chip failed to initialize with "Failed to get patch semaphore" on every boot.

This was the exact error from my previous post - the one that started this whole investigation. The irony: when I wrote the DMA ring fix that correctly defines both IRQ maps, I introduced the bug that prevented either from being selected correctly. The fix is one line - initialize at declaration:

```c
bool is_mt7927_hw = (pdev->device == 0x6639 || pdev->device == 0x7927);
```

## The Result

Both bands working on the ASUS ROG Crosshair X870E Hero:

```text
$ iw dev wlan0 link
Connected to xx:xx:xx:xx:xx:xx (on wlan0)
        SSID: MyNetwork-5G
        freq: 5200 (5 GHz)
        signal: -22 dBm
        tx bitrate: 1200.9 MBit/s HE-MCS 11 HE-NSS 2 HE-GI 0.8 80MHz
```

The initial change was 14 commits totaling 369 insertions and 20 deletions against the v6.19.3 mt76 source, distributed as per-concern patches in the DKMS tree. It has since grown to 18 patches (v2.4-1) against v6.19.6, plus a separate MT7902 WiFi 6E patch and Bluetooth patch - adding 320MHz support, ASPM fix, band_idx stability fix, and bug fixes from the community.

The patches modify twelve kernel source files: `mt76_connac.h`, `mt792x.h`, `mt792x_regs.h`, `mt792x_dma.c`, `mt7925/mt7925.h`, `mt7925/pci.c`, `mt7925/pci_mcu.c`, `mt7925/pci_mac.c`, `mt7925/init.c`, `mt7925/main.c`, `mt7925/mac.c`, `mt7925/mcu.c`. No new files. No new modules. The same mt7925e module, with branches for MT7927 hardware.

## What Happened Next

### 6GHz Works

A few days after publishing the initial package, [marcin-fm](https://github.com/openwrt/mt76/issues/927) tested on a TP-Link Archer TBE550E with an mt7995 6GHz AP. Results: 6GHz works. 160MHz channel width, 802.11be, PSK/SAE/OWE all work, and monitor mode works. Survey counters don't populate for 6GHz channels (firmware limitation), but the radio itself is fully operational across all three bands.

### The 320MHz Bug

marcin-fm also tested 320MHz on 6GHz - and it collapsed to 802.11n. The connection negotiated 320MHz initially, then fell back to 20MHz within seconds.

The root cause was in `mt7925_mcu_bss_rlm_tlv()`, which sends `UNI_BSS_INFO_RLM` to firmware with the operating bandwidth. It uses a switch on `chandef->width` to convert nl80211 channel widths to firmware constants:

```c
switch (chandef->width) {
case NL80211_CHAN_WIDTH_40:
    req->bw = CMD_CBW_40MHZ;
    break;
case NL80211_CHAN_WIDTH_80:
    req->bw = CMD_CBW_80MHZ;
    break;
case NL80211_CHAN_WIDTH_160:
    req->bw = CMD_CBW_160MHZ;
    break;
// NL80211_CHAN_WIDTH_320 is missing
default:
    req->bw = CMD_CBW_20MHZ;
    break;
}
```

`NL80211_CHAN_WIDTH_320` was missing from the switch, falling through to default - `CMD_CBW_20MHZ`. The firmware received 20MHz bandwidth for what should be 320MHz and negotiation collapsed.

This is a pre-existing upstream bug, not something introduced by the MT7927 patches. The mt7996 driver avoids it by using a lookup table (`mt76_connac_chan_bw()`) that already includes `CMD_CBW_320MHZ`. The mt7925 driver uses a hand-coded switch that was never updated for 320MHz. The fix is one case:

```c
case NL80211_CHAN_WIDTH_320:
    req->bw = CMD_CBW_320MHZ;
    break;
```

### The Missing PM Bit

Running a systematic self-review of all 14 commits against kernel subsystem patterns (wireless, PCI, power management) turned up a bug in the PM wake path.

`mt7927_dma_init()` sets `FW_DWLD_BYPASS_DMASHDL` (BIT(9) of GLO_CFG) during probe - it tells the DMA engine to bypass the download scheduler, which MT7927 needs for correct operation. But `mt792x_dma_enable()`, called on every PM wake, didn't restore this bit. Since every CLR_OWN triggers the ROM bootloader to reinitialize WFDMA (resetting GLO_CFG to defaults), the bit was lost after each power management cycle.

The existing `is_mt7927()` block in `mt792x_dma_enable()` already restored `ADDR_EXT_EN` and cleared `CSR_LBK_RX_Q_SEL_EN`. Adding `FW_DWLD_BYPASS_DMASHDL` to the same block completed the set. Without it, the DMA engine could silently malfunction after the first PM wake - the kind of bug that passes quick smoke tests but fails under sustained use.

### The ASPM Throttle

[cmspam](https://github.com/openwrt/mt76/issues/927) reported upload speeds capped at ~200 Mbps on an ASUS ROG Zephyrus G14 with an aftermarket MT7927 swap - while downloads hit 1.2 Gbps on the same connection. After systematic debugging, they found the root cause: PCIe ASPM L1 power saving.

```bash
echo 0 | sudo tee /sys/bus/pci/devices/0000:02:00.0/link/l1_aspm
```

That one command resolved the upload throttle immediately. With L1 disabled, throughput jumped to 1+ Gbps symmetrical on 160MHz HE-MCS 9/11 2x2.

The issue is specific to MT7927. The CONNINFRA power domain and WFDMA register access are unreliable when the PCIe link transitions in and out of L1 sleep state. Unlike MT7925 which handles L1 transitions gracefully (the driver adds a 2-3ms delay in `__mt792xe_mcu_drv_pmctrl()` to compensate), MT7927 depends on CONNINFRA being continuously accessible for CBTOP address remap, and its ROM reinitializes WFDMA on every CLR_OWN which can race with L1 transitions.

Other WiFi drivers have similar per-chip ASPM quirks - ath11k/ath12k disable L0s+L1 during firmware download, and rtw89 has separate per-chip L1 control.

The fix is one condition in `mt7925_pci_probe()`:

```c
if (mt7925_disable_aspm || is_mt7927_hw)
    mt76_pci_disable_aspm(pdev);
```

This unconditionally disables ASPM for MT7927 at probe time using the existing `mt76_pci_disable_aspm()` helper. It disables both L0s and L1 rather than just L1 - L0s power savings are negligible for a PCIe WLAN card, and using the existing helper avoids needing a separate L1-only code path with `CONFIG_PCIEASPM` fallback handling. After the disable, `mt76_pci_aspm_supported()` returns false so the 2-3ms delay in the DRV_OWN handshake is correctly skipped.

[Loong0x00](https://github.com/Loong0x00/mt7927) independently validated the fix: throughput increased 7x (6 MB/s to 46 MB/s) on their test setup.

### Performance

With all fixes applied, iperf3 between two wireless endpoints (MT7927 desktop and laptop, both on 5GHz WiFi, wired interfaces disabled):

| Direction | Throughput | TCP Retransmissions |
|-----------|-----------|---------------------|
| Upload (MT7927 -> laptop) | 316 Mbits/sec | 0 |
| Download (laptop -> MT7927) | 237 Mbits/sec | 45 |

MAC-layer stats: 12,764 TX retries out of 1,099,752 frames (1.16%), 0 TX failed. Signal strength -22 dBm, TX rate 1080 Mbps HE-MCS 10 80MHz 2x2. Throughput is limited by having WiFi on both endpoints (router is the bottleneck).

With the ASPM fix applied, other users achieved significantly higher throughput against wired endpoints:

| Tester | Setup | Throughput |
|--------|-------|-----------|
| cmspam | 160MHz HE-MCS 9/11 2x2, wired iperf3 server | 1+ Gbps up and down |
| PancakeTAS | 160MHz, BPI-R4 (MT7995AV) AP | ~1800 Mbps down, ~1600 Mbps up |
| syabro | 5GHz 80MHz MCS 11, rsync over SSH | ~720 Mbps sustained |
| Loong0x00 | 320MHz EHT-MCS 11 NSS 2, 6GHz, ASUS RT-BE92U | 2.01 Gbps (iperf3, 2.5GbE backhaul) |

A 2-hour stability test on the gateway showed 0 connection drops, 0 kernel errors, 0 beacon loss events. Signal ranged from -13 to -24 dBm across 240 samples. Suspend/wake (S3 deep sleep) tested across 3 consecutive cycles, all clean - the driver reconnects automatically after resume.

### TX Retransmissions: The Firmware Wall

Multiple users reported higher TX retransmission rates compared to MT7925 on the same AP. marcin-fm measured 7,902 retransmissions downloading over 802.11be vs near-zero on MT7925, same AP, same conditions.

I traced through the driver's TX power and calibration code looking for a driver-side cause. The MT7925/MT7927 driver delegates ALL TX power calibration to firmware - unlike mt7996, which reads per-channel power offsets from EEPROM. Only two EEPROM fields are used: `MT_EE_CHIP_ID` (0x000) and `MT_EE_HW_TYPE` (0xa71). Everything else comes from the firmware via `MCU_UNI_CMD(TXPOWER)`. There's no chip-specific branching in the calibration code, no per-channel adjustment the driver can make.

Conclusion: the TX retransmission issue is firmware-side. The driver has no knobs to turn. A firmware update from MediaTek is the likely fix, whenever that arrives.

### The Band Index Fix

After the initial release, users on 5GHz and 6GHz WPA3 networks hit `4WAY_HANDSHAKE_TIMEOUT` - the 4-way handshake would fail repeatedly, sometimes succeeding on the 3rd or 4th retry. 2.4GHz worked every time.

[marcin-fm](https://github.com/marcin-fm) and [Loong0x00](https://github.com/Loong0x00) independently discovered the root cause: the MT7927 firmware rejects `band_idx=0xff` (auto-select) in BSS configuration commands. MT7925 firmware handles auto-select, but MT7927 needs concrete values: 0 for 2.4GHz, 1 for 5/6GHz. Without the correct band index, the firmware silently drops EAPOL frames on the wrong band, causing the handshake timeout.

The fix adds explicit band index assignment in `mt7925_mcu_bss_basic_tlv()` when configuring the BSS for MT7927 hardware. A NetworkManager workaround (`connection.auth-retries=3`) also helps by retrying the handshake enough times to occasionally succeed, but the driver fix is the proper solution.

### Stale Pointer Fix

While reviewing the `change_vif_links` error path in the upstream driver, I found a pre-existing bug: stale pointers from the last loop iteration were used in the cleanup label instead of array-indexed access (`mconfs[link_id]` / `mlinks[link_id]`). This is a standalone upstream fix with a `Fixes:` tag, included in the series as patch 16.

### Planned Follow-Up Work

Two features are planned as follow-up patches after the base 18-patch series lands:

- **MLO (Multi-Link Operation)** - [Loong0x00](https://github.com/Loong0x00) contributed MLO support and verified STR dual-link traffic (5GHz+2.4GHz at 1.04 Gbps, 6GHz+2.4GHz at 1.55 Gbps) using hardware MIB counters. Sean Wang from MediaTek posted a [19-patch series](https://lore.kernel.org/linux-wireless/20260306232238.2039675-1-sean.wang@kernel.org/) fixing MLO link STA lifetime and error handling in mt7925 - switching to RCU lifetime, fixing WCID publish/teardown ordering, and adding host-side unwind on failure. MLO support will be submitted as a follow-up on top of that series.

- **mac_reset recovery** - [marcin-fm](https://github.com/marcin-fm) contributed a full DMA reinitialization on firmware crash. Has unguarded paths on mt7925 standalone that need fixing first.

### Known Limitations

| Issue | Status |
|-------|--------|
| TX retransmissions | Firmware-side, no driver fix possible |
| 6GHz survey counters | Not populated by MT7927 firmware |
| 6GHz MLO link | Passive scan + ML probe limitations prevent 6GHz link discovery (cfg80211/wpa_supplicant) |

### Community Testing

| Tester | Hardware | Result |
|--------|----------|--------|
| marcin-fm | TP-Link Archer TBE550E, Fedora 43 | Most thorough: 2.4/5/6GHz, 802.11ax/be, PSK/SAE/OWE, AP mode, monitor mode. Found 320MHz bug, band_idx fix (independent). |
| Loong0x00 | ASUS ROG Crosshair X870E Hero, Arch Linux | Reverse-engineered standalone driver, band_idx fix, ASPM validation (7x improvement), MLO dual-link verified (5G+2.4G STR 1.04 Gbps, 6G+2.4G STR 1.55 Gbps). 320MHz 6GHz: 2.01 Gbps. S3 suspend/resume works (~8s reconnect). |
| cmspam | ASUS ROG Zephyrus G14, NixOS | Found ASPM L1 root cause. 1+ Gbps symmetrical after fix. Daily-driving it. |
| PancakeTAS | ASUS ProArt X870E + BPI-R4 (MT7995AV) | 160MHz: ~1800 DL / ~1600 UL Mbps. Confirmed 320MHz broken (pre-fix). |
| syabro | Arch Linux, kernel 6.18.7, iwd | 5GHz 1201 Mbps MCS 11, ~720 Mbps rsync. BT works. First iwd user. |
| samutoljamo | Bazzite (Fedora Atomic) | Bazzite port. Working on compat patches for older kernels. |
| noobmaster777 | ASUS ProArt X870E, Ubuntu 24.04, kernel 6.17 | Works. |
| kerberos272 | Lenovo Legion Pro 7 16AFR10H | Works. Found shutdown hang (fixed in v2.1). |
| lubnin | ASUS X870E-E, Fedora 43, Secure Boot | New BT USB ID `13d3:3588` (IMC/Azurewave module). |
| tybautf | TP-Link TBE-550E, Freebox Ultra | 6GHz speed issue (known Freebox firmware bug). 4WAY_HANDSHAKE_TIMEOUT on kernel 6.17. |
| giosal | ASUS X870E-E, Ubuntu 26.04 | BT + WiFi confirmed. Maintains Ubuntu fork. 6GHz WPA3 ssid-not-found (regulatory timing). |
| BujuArena | ASUS ProArt X870E-Creator, CachyOS | Everything works out of the box. |
| cblinq | Gigabyte X870E Aorus Master X3D, EndeavourOS | WiFi confirmed. New PCI ID 14c3:7927 + BT USB 0489:e10f. |
| ryodeushii | Gigabyte Z790 Aorus Master X, CachyOS | WiFi on par with Intel AX210. |
| PMaff | ASUS ProArt X870E-Creator, openSUSE Tumbleweed | BT + WiFi confirmed. |
| ArielRosenfeld | ASUS ROG STRIX X870-F GAMING WIFI, Arch Linux | WiFi confirmed working. |
| Thex-Thex | Gigabyte Z790 Aorus Master X, Arch Linux | WiFi + BT confirmed. New BT USB ID 0489:e10f. |
| nitin88 | MSI X870E Ace Max, Fedora 43 | New BT USB ID 0489:e110, WiFi 14c3:7927. |
| IsaackRasmussen | ASUS ROG Strix X870-I | Testing in progress. |

### Upstream Status

The [18-patch WiFi series](https://lore.kernel.org/linux-wireless/20260306-mt7927-wifi-support-v1-0-c77e7445511d@jetm.me/) has been submitted to linux-wireless@ targeting wireless-next/main. Every new code path is gated behind `is_mt7927()` and doesn't modify behavior for existing MT7925 hardware. The series carries Tested-by trailers from 10 hardware testers across Arch Linux, CachyOS, EndeavourOS, Fedora, NixOS, openSUSE, and Ubuntu. MLO and mac_reset are planned as follow-up series once the base support lands.

The Bluetooth patches (v2 pending on linux-bluetooth@) are being revised per Luiz Dentz's request for per-device USB ID commits. The BT firmware has been submitted to linux-firmware via [MR !946](https://gitlab.com/kernel-firmware/linux-firmware/-/merge_requests/946).

## The Package

Everything is packaged as [`mediatek-mt7927-dkms`](https://aur.archlinux.org/packages/mediatek-mt7927-dkms) on the AUR. See the [repo README](https://github.com/jetm/mediatek-mt7927-dkms#readme) for install instructions, supported hardware, and troubleshooting. For Arch Linux:

```bash
pikaur -S mediatek-mt7927-dkms   # or yay/paru
```

The package handles both Bluetooth (MT6639 via USB) and WiFi (MT7925e via PCIe). It auto-downloads firmware from the ASUS CDN, pulls mt76 source from kernel.org, applies patches, and installs everything as a DKMS tree that rebuilds on kernel updates.

Community-maintained packages for other distributions:

- **NixOS**: [clemenscodes/linux-mt7927](https://github.com/clemenscodes/linux-mt7927) - NixOS module with WiFi and BT patches
- **NixOS**: [cmspam/mt7927-nixos](https://github.com/cmspam/mt7927-nixos) - NixOS flake, auto-tracks GitHub
- **Ubuntu**: [giosal/mediatek-mt7927-dkms](https://github.com/giosal/mediatek-mt7927-dkms) - install script for Ubuntu/Debian
- **Bazzite**: [samutoljamo/bazzite-mt7927](https://github.com/samutoljamo/bazzite-mt7927) - Bazzite (Fedora Atomic) port

Known hardware using MT7927:

| Hardware | BT USB ID | WiFi PCI ID |
|----------|-----------|-------------|
| ASUS ROG Crosshair X870E Hero | `0489:e13a` | `14c3:6639` |
| ASUS ProArt X870E-Creator WiFi | `13d3:3588` | `14c3:6639` |
| ASUS ROG Strix X870-I | - | `14c3:7927` |
| ASUS X870E-E | `13d3:3588` | `14c3:7927` |
| Gigabyte X870E Aorus Master X3D | `0489:e10f` | `14c3:7927` |
| Gigabyte Z790 AORUS MASTER X | `0489:e10f` | `14c3:7927` |
| Lenovo Legion Pro 7 16ARX9 | `0489:e0fa` | `14c3:7927` |
| Lenovo Legion Pro 7 16AFR10H | `0489:e0fa` | `14c3:7927` |
| TP-Link Archer TBE550E PCIe | `0489:e116` | `14c3:7927` |
| EDUP EP-MT7927BE M.2 | - | `14c3:7927` |
| Foxconn/Azurewave M.2 modules | - | `14c3:6639` |
| MSI X870E Ace Max | `0489:e110` | `14c3:7927` |
| ASUS ROG STRIX X870-F GAMING WIFI | - | `14c3:7927` |
| AMD RZ738 (MediaTek MT7927) | - | `14c3:0738` |

## Acknowledgments

[Loong0x00](https://github.com/Loong0x00/mt7927) - your reverse-engineered standalone driver was the key to everything. Without your documentation of the MT7927 DMA ring layout, CBInfra initialization sequence, and packed prefetch registers, this would have taken months instead of days. Also found the `is_mt7927_hw` uninitialized variable bug after v2.0, independently validated the ASPM fix, discovered the band_idx fix for 5/6GHz authentication, and verified MLO dual-link traffic with hardware MIB counters.

[marcin-fm](https://github.com/openwrt/mt76/issues/927) - firmware extraction tooling from the ASUS driver package, and the most thorough community tester. Confirmed 6GHz working, found the 320MHz BSS RLM bug, independently discovered the band_idx fix, and confirmed AP mode on all three bands. Caught the EAPOL patch regression (STA insertion failure) before it went upstream.

[cmspam](https://github.com/cmspam/mt7927-nixos) - found the ASPM L1 root cause on the Zephyrus G14, which led to the throughput fix in v2.1-5. Now daily-driving the MT7927 on NixOS.

[PancakeTAS](https://github.com/openwrt/mt76/issues/927) - confirmed 160MHz performance at 1800/1600 Mbps on a BPI-R4 (MT7995AV) AP, and independently confirmed the 320MHz bandwidth negotiation bug.

[Eadinator](https://github.com/openwrt/mt76/issues/927#issuecomment-3936022734) - ASUS CDN download automation via the TokenHQ API.

[clemenscodes](https://github.com/clemenscodes/linux-mt7927) - the unified Bluetooth patch that started this whole project, and now maintains a NixOS module.

[samutoljamo](https://github.com/samutoljamo/bazzite-mt7927) - Bazzite (Fedora Atomic) port, and working on compat patches for older kernel versions.

And everyone in [OpenWRT mt76 issue #927](https://github.com/openwrt/mt76/issues/927) who spent the past 16 months documenting hardware IDs, firmware formats, and register dumps - syabro, noobmaster777, kerberos272, lubnin, IsaackRasmussen, giosal, tybautf, and many others. Open source works because people share what they find, even when they can't finish the job themselves.

## References

- [Loong0x00/mt7927](https://github.com/Loong0x00/mt7927) - Reverse-engineered standalone MT7927 WiFi driver
- [ehausig/mt7927](https://github.com/ehausig/mt7927) - MT7927 register mapping and state machine documentation
- [OpenWRT mt76 issue #927](https://github.com/openwrt/mt76/issues/927) - MT7927 Linux support tracking (Oct 2024-)
- [mediatek-mt7927-dkms on GitHub](https://github.com/jetm/mediatek-mt7927-dkms#readme) - Source, install guide, supported hardware, and troubleshooting
- [mediatek-mt7927-dkms AUR package](https://aur.archlinux.org/packages/mediatek-mt7927-dkms) - DKMS package for Arch Linux
- [Stability test script](https://gist.github.com/jetm/b680a887778618f98632c8b5d5503c0a) - Non-disruptive overnight WiFi monitor
- [WiFi patch series on linux-wireless@](https://lore.kernel.org/linux-wireless/20260306-mt7927-wifi-support-v1-0-c77e7445511d@jetm.me/) - 18-patch upstream submission
- [Sean Wang's MLO fix series](https://lore.kernel.org/linux-wireless/20260306232238.2039675-1-sean.wang@kernel.org/) - MLO link lifetime and error handling fixes
- [Previous post: MT7927 WiFi - Wrong Driver, Wrong Chip, No Driver]({{< ref "/posts/mt7927-wifi-the-missing-piece" >}})
- [First post: Enabling MT7927 Bluetooth on Linux]({{< ref "/posts/enabling-mt7927-bluetooth-on-linux" >}})
