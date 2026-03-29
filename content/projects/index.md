---
title: "Projects"
description: "Upstream kernel contributions, open source tools, and published writing"
ShowToc: true
ShowReadingTime: false
TocOpen: true
---

## Upstream Kernel Contributions

Work currently in active upstream review or recently merged. All
verifiable via public mailing list archives.

### MediaTek MT7927 WiFi 7 - linux-wireless@

18-patch series adding full WiFi 7 support for the MT7927/MT6639
(Filogic 380) to the mt76/mt7925 driver. Covers chip ID helpers, PCI
device IDs, CBTOP remap, DMA initialization, hardware bring-up,
mac_reset recovery, and suspend/resume. Series at v4, addressing
feedback from Sean Wang (MediaTek). Community-tested across 10+
hardware platforms with 9 Tested-by tags (ASUS, Lenovo, Foxconn, AMD
RZ738).

- **Status:** v4, active review
- **Mailing list:** [linux-wireless@](https://lore.kernel.org/linux-wireless/?q=javier+tia)
- **Blog series:** [MT7927 WiFi on Linux: Making It Work](/blog/posts/mt7927-wifi-making-it-work/)

---

### MediaTek MT7927 Bluetooth 5.4 - linux-bluetooth@

8-patch series enabling MT6639 Bluetooth in btusb/btmtk: USB device
ID registration, hardware variant support, firmware section filtering
to prevent chip hang, and firmware naming corrections. Companion
firmware submitted to linux-firmware (MR !946, pipeline passing).

- **Status:** v2, active review
- **Mailing list:** [linux-bluetooth@](https://lore.kernel.org/linux-bluetooth/?q=javier+tia)
- **Blog post:** [MT7927 Bluetooth: From DKMS to Upstream](/blog/posts/mt7927-bluetooth-upstream-submission/)

---

### libcamera Simple Pipeline - AGC and AWB fixes

3-patch series (v4) fixing two bugs affecting all sensors with >8-bit
output on the Simple pipeline: a proportional AGC controller replacing
the fixed +/-10% bang-bang step (eliminates brightness flicker), and
an AWB statistics normalization fix correcting a bit-depth mismatch
that produced a ~9% green color cast. Reviewed by engineers from Red
Hat, Collabora, and Ideas on Board.

- **Status:** v4, active review
- **Mailing list:** [libcamera-devel@](https://lists.libcamera.org/pipermail/libcamera-devel/2026-March/057635.html)
- **Blog post:** [Intel IPU6 Webcam: From Proprietary Stack to Mainline](/blog/posts/ipu6-webcam-libcamera-on-linux/)

---

### UEFI Secure Boot - meta-arm (Yocto)

Complete UEFI Secure Boot implementation for ARM platforms upstreamed
into meta-arm: U-Boot key enrollment, systemd-boot signing, Linux
kernel image signing, a reusable sbsign BitBake class, OE-QA runtime
test cases, and GitLab CI integration. Accepted after 8 revision
cycles (Oct 2024).

- **Status:** Merged
- **Accepted by:** Jon Mason (ARM)

---

### OP-TEE / optee-client - meta-arm

Patches replacing static tee-supplicant service dependencies with
udev-based dynamic activation, resolving initialization failures on
platforms with multiple /dev/teepriv* devices.

- **Status:** Merged (2023)

---

## Open Source Tools

### mediatek-mt7927-dkms

DKMS package bridging out-of-tree MT7927 WiFi 7 + Bluetooth 5.4
patches to Arch Linux AUR users while upstream review is in progress.
Supports 10+ hardware variants with automated firmware extraction and
an 8-hour stability test script.

- **Repository:** [github.com/jetm/mediatek-mt7927-dkms](https://github.com/jetm/mediatek-mt7927-dkms)
- **Language:** Shell, Python

---

### jig

A Go TUI for git workflows built with Bubble Tea. Consolidates
interactive staging, hunk-level add, diff viewing, commit log
browsing, fixup commits, and interactive rebase into a single tool -
replacing forgit, diffnav, tig, and git-interactive-rebase-tool.
Ships with goreleaser, golangci-lint, and a 90% test coverage
threshold.

- **Repository:** [github.com/jetm/jig](https://github.com/jetm/jig)
- **Language:** Go

---

### glpkg

Python CLI for the GitLab Generic Package Registry, published on PyPI
as `glpkg-cli`. Fills a gap in the official `glab` CLI for package
registry uploads. Includes shell completion, GitHub Actions CI/CD, and
a comprehensive test suite.

- **Repository:** [github.com/jetm/glpkg](https://github.com/jetm/glpkg)
- **PyPI:** [glpkg-cli](https://pypi.org/project/glpkg-cli/)
- **Language:** Python

---

### devspec

Spec-driven development workflow engine for Claude Code. Manages the
full AI-assisted development lifecycle through a Claude Code MCP server
with 23 tools, 12 skills, 4 agent definitions, and 6 event hooks.
Implements TF-IDF spec drift detection, stagnation detection, and
topological task dependency sorting. ~1,245 tests across 58 files.

- **Repository:** [github.com/jetm/devspec](https://github.com/jetm/devspec)
- **Language:** Python

---

## Published Writing

### Linaro Engineering Blog

- [From "Replace It with Intel" to Upstream: Bringing MediaTek Bluetooth/WiFi 7 to Linux](https://www.linaro.org/blog/from-replace-it-with-intel-to-upstream-bringing-mediatek-bluetooth-wifi-7-to-linux/) - Mar 2026
- [5 Common Mistakes When Enabling Your Arm Device in the Automated Test Lab](https://www.linaro.org/blog/5-common-mistakes-when-enabling-your-arm-device-in-the-automated-test-lab/) - Mar 2026
- [Enabling Standard Linux on Arm: How ONELab Accelerates Interoperability and SystemReady Continuous Compliance](https://www.linaro.org/blog/enabling-standard-linux-on-arm-how-onelab-accelerates-interoperability-and-systemready-continuous-compliance/) - Mar 2026

### Personal Blog

In-depth articles on Linux kernel development, Yocto, embedded
security, and systems engineering. Covers the full MT7927 upstream
journey across 4 posts, Intel IPU6 mainline migration, DKMS packaging,
UEFI/QEMU workflows, and more.

[All posts](/blog/posts/)
