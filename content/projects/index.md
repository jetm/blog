---
title: "Projects & Contributions"
description: "Upstream kernel patches, open source tools, 200+ merged PRs/MRs across GitHub and GitLab, and published writing on Linux internals"
ShowToc: true
ShowWordCount: false
ShowReadingTime: false
TocOpen: true
---

Linux kernel patches under active upstream review, open source tools
used in production, 200+ merged pull requests and merge requests
across GitHub and GitLab, and published technical writing on systems
engineering. Everything listed here is public and verifiable.

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

3-patch series fixing two bugs affecting all sensors with >8-bit output
on the Simple pipeline: a proportional AGC controller replacing the
fixed +/-10% bang-bang step (eliminates brightness flicker), and an AWB
statistics normalization fix correcting a bit-depth mismatch that
produced a ~9% green color cast. Reviewed by engineers from Red Hat,
Collabora, and Ideas on Board. Merged 2026-05-07. A follow-up 2-patch
series (sumShift cleanup + OV2740 tuning file calibrated from Intel
AIQB binary) is in review.

- **Status:** Merged (May 2026)
- **Patchwork:** [v5 series](https://patchwork.libcamera.org/project/libcamera/list/?series=5913)
- **Follow-up:** [sumShift + OV2740 tuning](https://lists.libcamera.org/pipermail/libcamera-devel/2026-May/058643.html)
- **Follow-up blog:** [Extracting Sensor Calibration from Intel's AIQB Binary for libcamera](/blog/posts/ipu6-aiqb-calibration/)
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
firmware extraction and an 8-hour stability test script. Released v2.10-1 and
v2.11-1 (Apr 2026) tracking the upstream WiFi 7 patch series at v4.

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

### shipcheck

EU Cyber Resilience Act (CRA) compliance auditor for embedded Linux and Yocto builds.
Reads what your Yocto build emits - SBOMs, CVE scan output, signing artefacts, license
manifests - and reports whether the image is ready to ship. Produces a readiness score
and a multi-file CRA evidence dossier (Annex VII technical doc, Declaration of
Conformity, CVE report, license audit). Pilot 0001 (poky Scarthgap
core-image-minimal) validated the full check set against real bitbake output. v0.0.6
released May 2026.

- **Repository:** [github.com/jetm/shipcheck](https://github.com/jetm/shipcheck)
- **PyPI:** [shipcheck](https://pypi.org/project/shipcheck/)
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

## GitLab Contributions

Merged merge requests to Linaro, SOAFEE, and upstream projects on
GitLab, grouped by initiative.

### Trusted Substrate - GPIT CI/CD Platform

End-to-end CI/CD platform for Yocto-based Trusted Substrate images
with automated LAVA testing, OEQA result parsing, and email reporting.

- [gpit!29](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/29) - Propagate Poky info to downstream jobs (Oct 2025)
- [gpit!25](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/25) - generate-test-results.py: add missing comma (Sep 2025)
- [gpit!23](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/23) - Add --log-file input and two-pass OEQA parsing (Sep 2025)
- [gpit!19](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/19) - Revert python3-uswid in the image (Aug 2025)
- [gpit!17](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/17) - Install python3-uswid in the image (Aug 2025)
- [gpit!15](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/15) - Enhancements to GitLab CI and build process (Jul 2025)
- [gpit!14](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/14) - Make public gpit-genericarm64.img.xz (Jul 2025)
- [gpit!13](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/13) - Switch to arm64 architecture for kv260 (Jul 2025)
- [gpit!12](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/12) - Use meta-ts-ci-kas to build genericarm64 (Jul 2025)
- [gpit!10](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/10) - Add genericarm64 layer support and update LAVA testing (Jun 2025)
- [gpit!9](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/9) - Add new gpit genericarm64 image with SecureBoot (Jun 2025)
- [gpit!8](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/8) - Add dual pipeline for manual and automated Yocto testing (Jun 2025)
- [gpit!7](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/7) - Add Git describe row (Jun 2025)
- [gpit!6](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/6) - Add Poky Commit row in OEQA Test Summary (Jun 2025)
- [gpit!5](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/5) - Send email test results (May 2025)
- [gpit!4](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/4) - Export optee test suite (May 2025)
- [gpit!3](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/3) - Keep testexport tarball for LEDGE LAVA tests (May 2025)
- [gpit!2](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/2) - Publish LAVA test results with enhanced reporting (May 2025)
- [gpit!1](https://gitlab.com/Linaro/trustedsubstrate/gpit/-/merge_requests/1) - Integrate LAVA test failure detection, KV260 support, and kas migration (Apr 2025)

### Trusted Substrate - Testing & Secure Boot

UEFI Secure Boot testing infrastructure including EFI shell scripts,
capsule update validation, and container migration.

- [ts-testing!9](https://gitlab.com/Linaro/trustedsubstrate/ts-testing/-/merge_requests/9) - Migrate container image to Alpine (Jul 2025)
- [ts-testing!8](https://gitlab.com/Linaro/trustedsubstrate/ts-testing/-/merge_requests/8) - Compress QEMU bin files (Mar 2025)
- [ts-testing!7](https://gitlab.com/Linaro/trustedsubstrate/ts-testing/-/merge_requests/7) - Test valid and invalid capsule updates (Mar 2025)
- [ts-testing!6](https://gitlab.com/Linaro/trustedsubstrate/ts-testing/-/merge_requests/6) - Trigger meta-ts pipeline on master branch (Mar 2025)
- [ts-testing!5](https://gitlab.com/Linaro/trustedsubstrate/ts-testing/-/merge_requests/5) - Improve starting message (Mar 2025)
- [ts-testing!4](https://gitlab.com/Linaro/trustedsubstrate/ts-testing/-/merge_requests/4) - Add EFI shell test infrastructure (Jan 2025)
- [ts-testing!3](https://gitlab.com/Linaro/trustedsubstrate/ts-testing/-/merge_requests/3) - Move startup.nsh to binaries/ dir (Jan 2025)
- [ts-testing!2](https://gitlab.com/Linaro/trustedsubstrate/ts-testing/-/merge_requests/2) - Check ledge.efi exists before call (Jan 2025)
- [ts-testing!1](https://gitlab.com/Linaro/trustedsubstrate/ts-testing/-/merge_requests/1) - Add startup.nsh (Jan 2025)

### Trusted Substrate - meta-ledge-secure

Yocto security layer for UEFI Secure Boot, OP-TEE, disk encryption,
and EFI variable management.

- [meta-ledge-secure!121](https://gitlab.com/Linaro/trustedsubstrate/meta-ledge-secure/-/merge_requests/121) - Integrate UEFI Secure Boot from meta-arm (Dec 2024)
- [meta-ledge-secure!75](https://gitlab.com/Linaro/trustedsubstrate/meta-ledge-secure/-/merge_requests/75) - Fix tee-supplicant.service failure on AVA and xilinx-zcu102 (Sep 2023)
- [meta-ledge-secure!72](https://gitlab.com/Linaro/trustedsubstrate/meta-ledge-secure/-/merge_requests/72) - Remove kas traces (Aug 2023)
- [meta-ledge-secure!66](https://gitlab.com/Linaro/trustedsubstrate/meta-ledge-secure/-/merge_requests/66) - Detect mass storage health before encryption (Jun 2023)
- [meta-ledge-secure!62](https://gitlab.com/Linaro/trustedsubstrate/meta-ledge-secure/-/merge_requests/62) - Enable OS virtual console output on TRS (Jun 2023)
- [meta-ledge-secure!51](https://gitlab.com/Linaro/trustedsubstrate/meta-ledge-secure/-/merge_requests/51) - Fix efivar warning message (Mar 2023)
- [meta-ledge-secure!46](https://gitlab.com/Linaro/trustedsubstrate/meta-ledge-secure/-/merge_requests/46) - Resize root partition to take all available space (Mar 2023)

### SOAFEE Test Suite

Test framework for SOAFEE (Scalable Open Architecture for Embedded
Edge) covering container engine, k3s, Xen virtualization, OpenAD Kit,
and Linux ABI compliance. Includes TAP reporting, LAVA integration,
and documentation.

- [soafee-test-suite!31](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/31) - Add virtual test plan (Aug 2023)
- [soafee-test-suite!30](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/30) - Reset/set-up Synquacer Ethernet interface (Aug 2023)
- [soafee-test-suite!29](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/29) - Reset eth0 interface in Synquacer (Aug 2023)
- [soafee-test-suite!28](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/28) - Update README with current status (Jul 2023)
- [soafee-test-suite!27](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/27) - Report result in CI when passed and remove k3s requirement (Jul 2023)
- [soafee-test-suite!26](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/26) - Show journalctl results for diagnostics (Jul 2023)
- [soafee-test-suite!25](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/25) - Fix couple bugs (Jul 2023)
- [soafee-test-suite!24](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/24) - Add GitLab CI config for docs deployment (Jun 2023)
- [soafee-test-suite!23](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/23) - Document SOAFEE Test Suite - 2nd part (Jun 2023)
- [soafee-test-suite!22](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/22) - Add initial SOAFEE Test Suite documentation (Jun 2023)
- [soafee-test-suite!21](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/21) - Gather network information for diagnostics (Jun 2023)
- [soafee-test-suite!20](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/20) - Add initial Xen virtualization tests (Jun 2023)
- [soafee-test-suite!19](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/19) - Report overall results in TAP format (May 2023)
- [soafee-test-suite!18](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/18) - Convert TAP result into LAVA test cases (May 2023)
- [soafee-test-suite!17](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/17) - Disable OpenAD Kit tests (May 2023)
- [soafee-test-suite!16](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/16) - Remove redundant set -o errtrace and pipefail (May 2023)
- [soafee-test-suite!15](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/15) - Remove redundant set -o errexit commands (May 2023)
- [soafee-test-suite!14](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/14) - Introduce perf benchmarking OpenAD Kit tests (May 2023)
- [soafee-test-suite!13](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/13) - Add OpenAD Kit stress tests (Mar 2023)
- [soafee-test-suite!12](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/12) - Download and load Docker images from a mirror (Mar 2023)
- [soafee-test-suite!11](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/11) - Add OpenAD Kit VGA test (Mar 2023)
- [soafee-test-suite!10](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/10) - Use Docker images from GitLab (Feb 2023)
- [soafee-test-suite!9](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/9) - Add OpenAD Kit Docker image tests (Feb 2023)
- [soafee-test-suite!8](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/8) - Add SOAFEE Test Suite runtime (Feb 2023)
- [soafee-test-suite!7](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/7) - Add OpenAD Kit Linux ABI test (Feb 2023)
- [soafee-test-suite!6](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/6) - Allow to run test plans (Feb 2023)
- [soafee-test-suite!5](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/5) - Test docker socket creation (Feb 2023)
- [soafee-test-suite!4](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/4) - Add connection test for OpenAD Kit (Jan 2023)
- [soafee-test-suite!3](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/3) - Convert container subtests to individual tests (Jan 2023)
- [soafee-test-suite!2](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/2) - Port k3s-integration-tests into SOAFEE (Jan 2023)
- [soafee-test-suite!1](https://gitlab.com/soafee/soafee-test-suite/-/merge_requests/1) - Port container-engine-integration test to SOAFEE (Nov 2022)

### EWAOL (meta-ewaol)

Yocto layer for Edge Workload Abstraction and Orchestration Layer.
Recipe updates, kas configuration, and test suite integration.

- [meta-ewaol!43](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/43) - Update from styhead to walnascar (Aug 2025)
- [meta-ewaol!31](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/31) - Add myself as maintainer (Oct 2023)
- [meta-ewaol!30](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/30) - Update soafee-test-suite recipe to 247ebb6 (Sep 2023)
- [meta-ewaol!28](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/28) - Update soafee-test-suite recipe to f8bbf1b (Aug 2023)
- [meta-ewaol!27](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/27) - Hard-code Git hash in kas poky configuration (Aug 2023)
- [meta-ewaol!26](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/26) - Update soafee-test-suite recipe to a0eb006 (Jul 2023)
- [meta-ewaol!25](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/25) - Update soafee-test-suite recipe to 31bc53a (Jul 2023)
- [meta-ewaol!24](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/24) - Update soafee-test-suite recipe to a3b1004 (Jun 2023)
- [meta-ewaol!23](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/23) - Update soafee-test-suite recipe to 7931ea4 (Jun 2023)
- [meta-ewaol!22](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/22) - Update soafee-test-suite recipe to 31dcf0e (Jun 2023)
- [meta-ewaol!21](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/21) - Update soafee-test-suite recipe to e0f3797 (May 2023)
- [meta-ewaol!20](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/20) - Re-add soafee-test-suite recipe (May 2023)
- [meta-ewaol!18](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/18) - Remove soafee-test-suite recipe (May 2023)
- [meta-ewaol!17](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/17) - Skip downloading OpenAD Kit Demo docker image (Apr 2023)
- [meta-ewaol!16](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/16) - Update soafee-test-suite recipe to 9f2584f (Apr 2023)
- [meta-ewaol!14](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/14) - Update soafee-test-suite recipe to 8ef4c9a (Apr 2023)
- [meta-ewaol!13](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/13) - Fix n1sdp image build failure using kas (Apr 2023)
- [meta-ewaol!12](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/12) - Add stress-ng as soafee-test-suite run dependency (Mar 2023)
- [meta-ewaol!11](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/11) - Update soafee-test-suite to 3a77944 (Mar 2023)
- [meta-ewaol!8](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/8) - Update soafee-test-suite to 9c6275b (Feb 2023)
- [meta-ewaol!7](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/7) - Add Linux Test Project to EWAOL image (Feb 2023)
- [meta-ewaol!4](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/4) - Improve SOAFEE Test Suite report failures (Jan 2023)
- [meta-ewaol!2](https://gitlab.com/soafee/ewaol/meta-ewaol/-/merge_requests/2) - Add soafee-test-suite recipe (Nov 2022)
- [meta-ewaol-machine!3](https://gitlab.com/soafee/ewaol/meta-ewaol-machine/-/merge_requests/3) - avadp: Add efivar command (Jan 2023)
- [meta-ewaol-machine!2](https://gitlab.com/soafee/ewaol/meta-ewaol-machine/-/merge_requests/2) - Set up bitbake to use local hackbox2 mirror (Nov 2022)

### Blueprints CI

LAVA-based CI/CD infrastructure for Yocto image testing on ARM
hardware (Synquacer, RockPi4, KV260, AVA). Covers test plans, capsule
updates, image merging, and SOAFEE integration.

- [ci!190](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/190) - Refresh README based on transition to kas (Feb 2025)
- [ci!189](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/189) - Match EFI test devices with LAVA devices (Feb 2025)
- [ci!182](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/182) - Fix EFI test for rock-pi-4b (Feb 2025)
- [ci!181](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/181) - Transition to kas build tool and EFI testing (Feb 2025)
- [ci!180](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/180) - Make EFI test to use dynamic CI images (Jan 2025)
- [ci!176](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/176) - Add EFI tests to test plan (Jan 2025)
- [ci!172](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/172) - Add Git Merge branch in kas conf (Jan 2025)
- [ci!171](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/171) - Replace refspec with branch in kas (Jan 2025)
- [ci!170](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/170) - Add kas build meta-ts (Dec 2024)
- [ci!168](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/168) - Clean BOOT and Capsules UEFI vars in rockpi4b (Dec 2024)
- [ci!146](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/146) - Power-cycle kv260 before updating capsule (Mar 2024)
- [ci!144](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/144) - Use RockPi4 invalid capsule from ci-artifacts repo (Feb 2024)
- [ci!143](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/143) - Enable LAVA capsule update testing for kv260 (Feb 2024)
- [ci!133](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/133) - Replace wait-on-systemd-units test with ping (Sep 2023)
- [ci!132](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/132) - Conditionally run virtual soafee-test-suite test plan (Sep 2023)
- [ci!127](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/127) - Fix synquacer romramfw URL (Jul 2023)
- [ci!126](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/126) - Refactor wildcard expression to pick TS/TRS images (Jul 2023)
- [ci!121](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/121) - Add Xen Guest Images tarball (Jun 2023)
- [ci!118](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/118) - Integrate soafee-test-suite results into LAVA (May 2023)
- [ci!117](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/117) - Re-enable soafee-test-suite in AVA (May 2023)
- [ci!116](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/116) - Add meta-ewaol tests YAML file (May 2023)
- [ci!112](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/112) - Remove soafee-test-suite in AVA CI Test Plan (May 2023)
- [ci!108](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/108) - Increase run timeout to 60m for SOAFEE Test Suite (Apr 2023)
- [ci!105](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/105) - Resize to 10 GB qemu image disk (Apr 2023)
- [ci!97](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/97) - Improve MR workflow with a template (Apr 2023)
- [ci!91](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/91) - Rewrite ts-merge-images.sh to support multiple partitions (Mar 2023)
- [ci!90](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/90) - Use ts-merge-images.sh from BP CI repo (Mar 2023)
- [ci!89](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/89) - Add ts-merge-images.sh script (Mar 2023)
- [ci!78](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/78) - Run soafee-test-suite-setup before soafee-test-suite (Mar 2023)
- [ci!67](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/67) - Depend on the required systemd units (Feb 2023)
- [ci!63](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/63) - Wait for systemd is-running state before tests (Feb 2023)
- [ci!61](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/61) - Add test duration time in SOAFEE Test Suite (Feb 2023)
- [ci!40](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/40) - Make soafee-test-suite report on TAP format (Jan 2023)
- [ci!28](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/28) - Run SOAFEE Test Suite in AVA (Dec 2022)
- [ci!20](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/20) - Add SOAFEE Test Suite into LAVA (Dec 2022)
- [ci!8](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/8) - Fix how to run the test locally (Oct 2022)
- [ci!5](https://gitlab.com/Linaro/blueprints/ci/-/merge_requests/5) - Test UEFI Secure Basics (Oct 2022)

### Trusted Reference Stack

Yocto distribution layer for ARM Trusted Reference Stack. Build
system improvements, QEMU tooling, mirror setup, and recipe updates.

- [trs!216](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/216) - Add SBSIGN_KEYS_DIR var (Dec 2024)
- [trs!178](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/178) - Update missing python3-pexpect as build prerequisites (Feb 2024)
- [trs!134](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/134) - Add kirkstone to virtualization layer (Jan 2023)
- [trs!132](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/132) - Replace MR template with a simpler version (Apr 2023)
- [trs!130](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/130) - Remove kas traces from documentation (Aug 2023)
- [trs!129](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/129) - Replace ethtool with mii-tool (Aug 2023)
- [trs!127](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/127) - Update soafee-test-suite recipe to f8bbf1b (Aug 2023)
- [trs!126](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/126) - Add ethtool to TRS image (Aug 2023)
- [trs!121](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/121) - Remove kas from python-prereqs and update docs (Aug 2023)
- [trs!120](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/120) - Update soafee-test-suite recipe to a0eb006 (Jul 2023)
- [trs!117](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/117) - Update soafee-test-suite recipe to 31bc53a (Jul 2023)
- [trs!116](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/116) - Avoid resize image if calling make run more than once (Jul 2023)
- [trs!114](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/114) - Update soafee-test-suite recipe to a3b1004 (Jun 2023)
- [trs!113](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/113) - Update soafee-test-suite recipe to 7931ea4 (Jun 2023)
- [trs!111](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/111) - Update soafee-test-suite recipe to 31dcf0e (Jun 2023)
- [trs!106](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/106) - Update soafee-test-suite recipe to e0f3797 (May 2023)
- [trs!103](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/103) - Move from ewaol-machine to trs (May 2023)
- [trs!102](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/102) - Update soafee-test-suite recipe to d0b19d8 (May 2023)
- [trs!94](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/94) - Add MR template (Mar 2023)
- [trs!92](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/92) - Resize qemu image in runtime (Mar 2023)
- [trs!73](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/73) - Disable k3s systemd service (Feb 2023)
- [trs!72](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/72) - Ignore busybox-initrd recipe from meta-virtualization (Feb 2023)
- [trs!46](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/46) - Allow to set the number of threads in local.conf (Jan 2023)
- [trs!44](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/44) - Add kirkstone to virtualization layer (Jan 2023)
- [trs!32](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/32) - Set up bitbake to use local mirror (Nov 2022)
- [trs!29](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/29) - Add gen-mirror-tar target (Nov 2022)
- [trs!22](https://gitlab.com/Linaro/trusted-reference-stack/trs/-/merge_requests/22) - Update DL_DIR and SSTATE_DIR in local.conf (Nov 2022)
- [trs-manifest!138](https://gitlab.com/Linaro/trusted-reference-stack/trs-manifest/-/merge_requests/138) - Change to meta-openembedded from GitHub (Feb 2025)
- [trs-manifest!134](https://gitlab.com/Linaro/trusted-reference-stack/trs-manifest/-/merge_requests/134) - Update to latest meta-xilinx master hash (Dec 2024)
- [trs-manifest!133](https://gitlab.com/Linaro/trusted-reference-stack/trs-manifest/-/merge_requests/133) - Update manifest for UEFI Secure Boot (Nov 2024)
- [trs-manifest!27](https://gitlab.com/Linaro/trusted-reference-stack/trs-manifest/-/merge_requests/27) - Change to master from upstream meta-virtualization (Feb 2023)
- [trs-manifest!18](https://gitlab.com/Linaro/trusted-reference-stack/trs-manifest/-/merge_requests/18) - Update meta-virtualization to langdale (Jan 2023)
- [poky!2](https://gitlab.com/Linaro/trusted-reference-stack/poky/-/merge_requests/2) - go: update to v1.19.4 (Feb 2023)

### ONELab & LAVA

ONELab documentation, container tooling, and LAVA appliance firmware
flashing.

- [onelab-user-guide!7](https://gitlab.com/Linaro/onelab/documentation/customer/onelab/onelab-user-guide/-/merge_requests/7) - Add Sprinto SLScan stage and job (Dec 2025)
- [artifactory!7](https://gitlab.com/Linaro/onelab/tools/artifactory/-/merge_requests/7) - Refactor multi-architecture image building process (Dec 2025)
- [artifactory!6](https://gitlab.com/Linaro/onelab/tools/artifactory/-/merge_requests/6) - Improve cabinet-builder image and build workflow (Dec 2025)
- [firmwares!17](https://gitlab.com/Linaro/lava/appliance/firmwares/-/merge_requests/17) - am62pxx-flash-fw: Install tio in LAA and validate version (Apr 2025)
- [firmwares!11](https://gitlab.com/Linaro/lava/appliance/firmwares/-/merge_requests/11) - am62pxx-sk: Support flashing FW using UART (Mar 2025)
- [firmwares!7](https://gitlab.com/Linaro/lava/appliance/firmwares/-/merge_requests/7) - am62pxx-sk: Change Boot Mode to eMMC (Jan 2025)

### Automotive SOAFEE Test Suite (legacy)

Early SOAFEE test suite work before migration to the dedicated
soafee/ namespace.

- [soafee-test-suite!3](https://gitlab.com/Linaro/blueprints/automotive/soafee-test-suite/-/merge_requests/3) - Repository/packaging improvements (Nov 2022)
- [soafee-test-suite!2](https://gitlab.com/Linaro/blueprints/automotive/soafee-test-suite/-/merge_requests/2) - Merge meta-ewaol (Oct 2022)

---

### Linaro Engineering Blog

- [From "Replace It with Intel" to Upstream: Bringing MediaTek Bluetooth/WiFi 7 to Linux](https://www.linaro.org/blog/from-replace-it-with-intel-to-upstream-bringing-mediatek-bluetooth-wifi-7-to-linux/) - Mar 2026
- [5 Common Mistakes When Enabling Your Arm Device in the Automated Test Lab](https://www.linaro.org/blog/5-common-mistakes-when-enabling-your-arm-device-in-the-automated-test-lab/) - Mar 2026
- [Enabling Standard Linux on Arm: How ONELab Accelerates Interoperability and SystemReady Continuous Compliance](https://www.linaro.org/blog/enabling-standard-linux-on-arm-how-onelab-accelerates-interoperability-and-systemready-continuous-compliance/) - Mar 2026

### Press Coverage

- [MediaTek MT7927 WiFi 7 + Bluetooth 5.4 Linux Support Coming Together](https://www.phoronix.com/news/MediaTek-MT7927-WiFi-Linux) - Phoronix, Mar 2026

### Personal Blog

In-depth articles on Linux kernel development, Yocto, embedded
security, and systems engineering.

- [Extracting Sensor Calibration from Intel's AIQB Binary for libcamera](/blog/posts/ipu6-aiqb-calibration/) - May 2026
- [Yocto build tunables and their hidden costs](/blog/posts/yocto-build-tunables-and-their-hidden-costs/) - May 2026
- [When You're Fired, Your Next Job Is Finding a Job](/blog/posts/when-youre-fired-your-next-job-is-finding-a-job/) - Apr 2026
- [Auditing your Yocto build for CRA compliance](/blog/posts/auditing-your-yocto-build-for-cra-compliance/) - Apr 2026
- [MT7927 Bluetooth: From DKMS to Upstream](/blog/posts/mt7927-bluetooth-upstream-submission/) - Mar 2026
- [MT7927 WiFi on Linux: Making It Work](/blog/posts/mt7927-wifi-making-it-work/) - Mar 2026
- [Intel IPU6 Webcam on Linux: From Proprietary Stack to Mainline](/blog/posts/ipu6-webcam-libcamera-on-linux/) - Feb 2026
- [MT7927 WiFi on Linux: Wrong Driver, Wrong Chip, No Driver](/blog/posts/mt7927-wifi-the-missing-piece/) - Feb 2026
- [Enabling MediaTek MT7927 Bluetooth on Linux: A 15-Month Journey](/blog/posts/enabling-mt7927-bluetooth-on-linux/) - Feb 2026
- [Building a Bootable Windows USB from Linux for Firmware Updates](/blog/posts/building-bootable-windows-usb-from-linux-for-firmware-updates/) - Feb 2026

[All posts](/blog/posts/)
