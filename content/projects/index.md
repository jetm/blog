---
title: "Projects"
description: "Upstream kernel contributions, open source tools, GitHub contributions, and published writing"
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
- **Language:** C

---

### MediaTek MT7927 Bluetooth 5.4 - linux-bluetooth@

8-patch series enabling MT6639 Bluetooth in btusb/btmtk: USB device
ID registration, hardware variant support, firmware section filtering
to prevent chip hang, and firmware naming corrections. Companion
firmware submitted to linux-firmware (MR !946, pipeline passing).

- **Status:** v2, active review
- **Mailing list:** [linux-bluetooth@](https://lore.kernel.org/linux-bluetooth/?q=javier+tia)
- **Blog post:** [MT7927 Bluetooth: From DKMS to Upstream](/blog/posts/mt7927-bluetooth-upstream-submission/)
- **Language:** C

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
- **Language:** C++

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
Packages for Debian and Fedora. Supports 10+ hardware variants with automated
firmware extraction and an 8-hour stability test script.

- **Repository:** [github.com/jetm/mediatek-mt7927-dkms](https://github.com/jetm/mediatek-mt7927-dkms)
- **Language:** Shell, C, Python

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

## Open Source Contributions

Merged pull requests to third-party projects on GitHub, grouped by
domain.

### Firmware & Security

- [fwupd#9105](https://github.com/fwupd/fwupd/pull/9105) - Enhance firmware metadata generation in firmware_packager (Aug 2025)
- [edk2#10844](https://github.com/tianocore/edk2/pull/10844) - Fix proper return after capsule update in CapsuleApp (Mar 2025)
- [mbedtls#9105](https://github.com/Mbed-TLS/mbedtls/pull/9105) - Add Host header to ssl_client2 HTTP GET request (Oct 2024)
- [mbedtls#9118](https://github.com/Mbed-TLS/mbedtls/pull/9118) - Backport ssl_client2 Host header fix to 3.6 (Oct 2024)
- [meta-secure-core#76](https://github.com/Wind-River/meta-secure-core/pull/76) - Add gen-sbkeys.bb recipe for UEFI Secure Boot key generation (Oct 2024)
- [openwrt#13512](https://github.com/openwrt/openwrt/pull/13512) - Enable KASLR in kernel 6.1 for armsr/armv8 (Oct 2023)
- [test-definitions#385](https://github.com/Linaro/test-definitions/pull/385) - Add Secure Boot Enabled test definition (Nov 2022)

### Developer Tools & AI

- [summarize#109](https://github.com/steipete/summarize/pull/109) - Support CLI models in daemon chat and agent endpoints (Mar 2026)
- [difi#30](https://github.com/xguot/difi/pull/30) - Fix CalculateFileLine off-by-one and header misparse (Feb 2026)
- [llmswap#9](https://github.com/sreenathmmenon/llmswap/pull/9) - Refactor CLI entrypoint and update README (Dec 2025)
- [gibr#55](https://github.com/ytreister/gibr/pull/55) - Add uv instructions for optional dependencies (Nov 2025)
- [conform.nvim#579](https://github.com/stevearc/conform.nvim/pull/579) - Add commitmsgfmt formatter (Nov 2024)
- [LazyVim#1229](https://github.com/LazyVim/LazyVim/pull/1229) - Fix yaml lang TypeError on undefined length (Jul 2023)
- [SchemaStore.nvim#19](https://github.com/b0o/SchemaStore.nvim/pull/19) - Document yaml option to fix LSP completion (Jul 2023)
- [harvey#10](https://github.com/architv/harvey/pull/10) - Fix crash with requests v2.8.1 (Nov 2015)

### Vim/Neovim Ecosystem

- [SpaceVim#1725](https://github.com/wsdjeg/SpaceVim/pull/1725) - Add [SPC]bb map to list all buffers in fzf layer (Jun 2018)
- [gen_tags.vim#45](https://github.com/jsfaint/gen_tags.vim/pull/45) - Introduce g:gen_gtags#gtags_bin and gtags_opts (May 2018)
- [SpaceVim#724](https://github.com/wsdjeg/SpaceVim/pull/724) - Make explicit a number of lines is required (Jul 2017)
- [SpaceVim#659](https://github.com/wsdjeg/SpaceVim/pull/659) - Add comment space (Jun 2017)
- [SpaceVim#578](https://github.com/wsdjeg/SpaceVim/pull/578) - Add documentation about colorscheme (May 2017)
- [SpaceVim#538](https://github.com/wsdjeg/SpaceVim/pull/538) - Fix exclude syntax (-g) for rg util (May 2017)
- [SpaceVim#455](https://github.com/wsdjeg/SpaceVim/pull/455) - Add \<SPC fW\> mapping (Apr 2017)
- [SpaceVim#436](https://github.com/wsdjeg/SpaceVim/pull/436) - Use on_event instead of deprecated on_i (Apr 2017)
- [spf13-vim#768](https://github.com/spf13/spf13-vim/pull/768) - Remove map semicolon to colon (May 2015)

### Build Systems & Infrastructure

- [Ceedling#58](https://github.com/ThrowTheSwitch/Ceedling/pull/58) - Surround YAML and C code with fenced code blocks (Mar 2016)
- [Ceedling#57](https://github.com/ThrowTheSwitch/Ceedling/pull/57) - Normalize line endings, convert CRLF to LF (Mar 2016)
- [build-tools-cpp#58](https://github.com/fstiewitz/build-tools-cpp/pull/58) - Swap mentions of u/o key maps call command (Feb 2016)
- [workrave#50](https://github.com/rcaelers/workrave/pull/50) - Build with C++11 if gtkmm >= 3.18.0 (Oct 2015)
- [mkdocs#584](https://github.com/mkdocs/mkdocs/pull/584) - Add copyright footer for readthedocs theme (Jun 2015)

### Documentation & Community

- [ec#1](https://github.com/chojs23/ec/pull/1) - Add AUR installation options (Feb 2026)
- [difi#16](https://github.com/xguot/difi/pull/16) - Add AUR installation instructions for Arch Linux (Feb 2026)
- [oelint-adv#28](https://github.com/priv-kweihmann/oelint-adv/pull/28) - Add Arch Linux install instruction (Oct 2019)
- [bic#22](https://github.com/hexagonal-sun/bic/pull/22) - Add Arch Linux install instruction (Oct 2019)
- [oelint-adv#20](https://github.com/priv-kweihmann/oelint-adv/pull/20) - Ignore PyCharm & VSCode, apply PEP8 rules (Aug 2019)
- [docker#28](https://github.com/OpenGrok/docker/pull/28) - Remove data and src symlinks (Mar 2019)
- [bashew#16](https://github.com/pforret/bashew/pull/16) - Format all shell codes with shfmt (Nov 2022)
- [reproc#5](https://github.com/daandemeyer/reproc/pull/5) - Reword the different ways to install reproc (Sep 2018)
- [yank#32](https://github.com/mptre/yank/pull/32) - Add Arch Linux install instruction (Jan 2017)
- [dtags#5](https://github.com/joowani/dtags/pull/5) - Keep previous $IFS, do not lose it (Mar 2016)
- [pac-info#1](https://github.com/shaleh/pac-info/pull/1) - Add example how to use gen-proxy-env (Aug 2016)

---


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
