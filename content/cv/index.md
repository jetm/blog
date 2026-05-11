---
title: "Javier Tia - CV"
description: "Senior Systems Engineer specializing in upstream Linux kernel development, embedded Linux, UEFI Secure Boot, and CI/CD infrastructure."
ShowToc: false
ShowReadingTime: false
ShowWordCount: false
ShowBreadCrumbs: false
ShowPostNavLinks: false
layout: "cv"
outputs:
  - HTML
---

## Summary

Senior Systems Engineer with 15+ years of production embedded software experience,
specializing in upstream Linux kernel development, Yocto/OpenEmbedded, embedded
security (UEFI Secure Boot, OP-TEE), and CI/CD infrastructure for hardware
platforms. Active upstream contributor with recent merges into bluetooth-next
(targeting Linux 7.1/7.2) and libcamera, and active patch series under review on
linux-wireless@, linux-firmware, and Yocto. Comfortable working fully remotely
across time zones. Open to consulting engagements and full-time remote roles.

## Technical Skills

**Kernel & firmware:** Linux kernel development, DKMS, U-Boot, UEFI/Secure Boot,
OP-TEE, device drivers (WiFi/mac80211/cfg80211, Bluetooth/btusb/btmtk, PCIe,
camera/libcamera)

**Build systems:** Yocto/OpenEmbedded, BitBake, kas, (meta-arm, meta-openembedded,
meta-secure-core), Make, CMake, Meson, Autotools, repo tool

**CI/CD & testing:** LAVA, SQUAD, TuxSuite, GitLab CI, GitHub Actions

**Languages:** C, C++, Go, Python, Shell, Perl

**Security:** UEFI Secure Boot, measured boot, TPM, sbsign, EFI variable
management (libefivar)

**Backend:** Django, REST APIs, shell automation

**Wireless:** mac80211, cfg80211, mt76 driver family, btusb/btmtk Bluetooth stack

**Platforms:** ARM/AArch64/ARMv8, x86-64, RISC-V, QEMU, Raspberry Pi, NXP i.MX,
Qualcomm, MediaTek, Xilinx/AMD KV260, Synquacer

**Embedded:** BSP development, board bring-up, device tree (DTS/DTB),
cross-compilation, UART/I2C/SPI, systemd, udev, initramfs

## Experience

### Linaro - Senior Systems Engineer
**Sep 2022 – Mar 2026 | Remote**

Linaro is an open-source collaborative engineering organization focused on
Arm-based hardware and software. Worked across upstream kernel enablement,
embedded security, CI/CD infrastructure, and platform bring-up for ARM-based
reference stacks used by Linaro member companies.

**Upstream Kernel & Firmware**
- Authored 18-patch WiFi 7 series for MediaTek MT7927/MT6639 (Filogic 380) in the
  mt76/mt7925 driver (linux-wireless@, v4) - community-tested across 10+ hardware
  platforms with 9 Tested-by tags from ASUS, Lenovo, Foxconn, and AMD
- Authored 8-patch Bluetooth series enabling MT6639 in btusb/btmtk; merged into
  bluetooth-next, targeting mainline Linux 7.1/7.2; companion firmware submitted
  to linux-firmware
- Authored 3-patch libcamera AGC/AWB fixes for Simple pipeline sensors with >8-bit
  output (libcamera-devel@) - merged May 2026; follow-up 2-patch series (OV2740
  tuning file) in review
- Maintained Linux kernel stable branches for ARM embedded targets (KV260,
  RockPi4, Synquacer, AVA, TI SK-AM62P-LP, NXP i.MX 8M, Raspberry Pi, genericarm64):
  CVE backports, stability patches, regular release cadence
- Contributed upstream patches to u-boot/u-boot for ARM platform bring-up, Secure
  Boot chain configuration, and UEFI key management

**Embedded Security**
- Designed and upstreamed complete UEFI Secure Boot implementation into meta-arm:
  U-Boot key enrollment, systemd-boot + kernel signing, sbsign BitBake class,
  OE-QA runtime test cases, GitLab CI - merged after 8 revision cycles (Oct 2024,
  reviewed by Jon Mason, ARM)
- Authored OP-TEE/optee-client patches: udev-based tee-supplicant activation
  replacing static dependencies, resolving failures on multi-device
  /dev/teepriv* platforms (merged 2023)
- Contributed to meta-ledge-secure Yocto security layer: Secure Boot integration,
  disk encryption, and EFI variable management

**CI/CD Infrastructure**
- Built GPIT - end-to-end GitLab CI/CD for Yocto Trusted Substrate images with
  automated LAVA hardware testing, OEQA result parsing, two-pass log analysis, and
  email reporting across ARM targets (KV260, RockPi4, Synquacer, AVA,
  genericarm64) - 19 merged MRs
- Built and maintained LAVA-based CI/CD for embedded Linux testing across ARM
  hardware (Synquacer, RockPi4, KV260, AVA): test plans, capsule update pipelines,
  image merging scripts, GitLab CI integration - 40+ merged MRs
- Integrated ARM development boards into LAVA, SQUAD, and TuxSuite pipelines,
  increasing automated hardware test coverage and reducing manual validation cycles
- Built SOAFEE test framework: container engine, k3s, Xen virtualization, OpenAD
  Kit, and Linux ABI compliance testing with TAP reporting and LAVA integration -
  30+ merged MRs

**Yocto/OpenEmbedded**
- Owned the Yocto/OE build system delivering all software and kernels for
  ARM-based embedded Linux products - Trusted Substrate and SOAFEE reference
  images across KV260, RockPi4, Synquacer, AVA, TI SK-AM62P-LP, NXP i.MX 8M,
  Raspberry Pi, and genericarm64 - managing layers across meta-openembedded,
  meta-arm, meta-secure-core, and meta-arm-bsp
- Established reproducible BitBake configurations and sstate-cache strategies,
  reducing build failures across the team
- Active patch reviewer and contributor across meta-openembedded, meta-arm, and
  meta-secure-core

**Backend & Tooling**
- Contributed Django REST API features to ONELab (Linaro's edge/IoT
  interoperability testing platform): automated GitLab CI deployment pipelines and
  shell tooling, reducing manual device onboarding effort
- Integrated EFI variable tooling (rhboot/efivar) into embedded firmware stacks,
  enabling runtime EFI variable access from Linux user-space on ARM platforms

---

### Open Source Engineer (Independent)
**Mar 2026 – Present | Remote**

Continued upstream kernel development and open source tooling after leaving Linaro.

**Upstream & Open Source**
- Authored libcamera follow-up series: `sumShift` cleanup and OV2740 sensor tuning file
  calibrated from Intel AIQB binary (libcamera-devel@, under review)
- Released mediatek-mt7927-dkms v2.10-1 and v2.11-1 tracking the upstream WiFi 7 patch series
- Built shipcheck - EU Cyber Resilience Act (CRA) compliance auditor for Yocto/OE builds:
  reads SBOM, CVE scan output, code-signing artefacts, and license manifests; produces
  readiness score and multi-file CRA evidence dossier. v0.0.6 released May 2026, published
  on PyPI. Contributed upstream Yocto SPDX 3.0 patch series (openembedded-core) as part of
  gap analysis

**Technical Writing**
- [Extracting Sensor Calibration from Intel's AIQB Binary for libcamera](/blog/posts/ipu6-aiqb-calibration/) - May 2026
- [Yocto build tunables and their hidden costs](/blog/posts/yocto-build-tunables-and-their-hidden-costs/) - May 2026
- [Auditing your Yocto build for CRA compliance](/blog/posts/auditing-your-yocto-build-for-cra-compliance/) - Apr 2026
- [When You're Fired, Your Next Job Is Finding a Job](/blog/posts/when-youre-fired-your-next-job-is-finding-a-job/) - Apr 2026

---

### Hewlett Packard Enterprise / Aruba - Systems Engineer
**Mar 2011 – Sep 2022 | Remote**

11 years of production embedded software engineering on HPE/Aruba network switches
running embedded Linux. Worked across the full firmware stack for high-availability
enterprise networking hardware.

- Participated in large-scale re-architecture effort to modularize and modernize
  HPE/Aruba switch firmware - production-grade embedded Linux across a fleet of
  enterprise networking devices
- Developed and maintained embedded Linux BSPs for network switch platforms: kernel
  configuration, driver integration, board bring-up, and system-level testing
- Worked on firmware build systems, CI integration, and release engineering for
  embedded switch software
- Contributed to cross-functional efforts spanning kernel, user-space, and
  management plane components in a large, distributed engineering organization

## Upstream Contributions

| Project | Contribution | Status |
|---|---|---|
| linux-wireless@ | MT7927 WiFi 7 - 18-patch series (chip ID, PCI IDs, DMA init, mac_reset, suspend/resume) | v4, active review |
| linux-bluetooth@ | MT7927 Bluetooth - 8-patch series (USB ID, firmware filtering, naming) | Merged into bluetooth-next (targeting Linux 7.1/7.2) |
| libcamera-devel@ | Simple pipeline AGC/AWB fixes (>8-bit sensor support) | Merged May 2026 |
| libcamera-devel@ | sumShift cleanup + OV2740 sensor tuning file (Intel AIQB extraction) | Under review |
| linux-firmware | MT6639 Bluetooth firmware (MR!946) | Under review |
| meta-arm (Yocto) | UEFI Secure Boot - full implementation (v8) | Merged Oct 2024 |
| meta-arm (Yocto) | OP-TEE udev-based tee-supplicant activation | Merged 2023 |
| edk2 | CapsuleApp proper return after capsule update | Merged Mar 2025 |
| fwupd | Firmware metadata generation enhancement | Merged Aug 2025 |
| OpenWrt | KASLR for armsr/armv8 kernel 6.1 | Merged Oct 2023 |
| mbedtls | Host header fix in ssl_client2 (3.x + 3.6 backport) | Merged Oct 2024 |

Full list of 200+ merged PRs/MRs: [jetm.github.io/blog/projects](https://jetm.github.io/blog/projects/)

## Projects & Tooling

**[mediatek-mt7927-dkms](https://github.com/jetm/mediatek-mt7927-dkms)** - DKMS
package bridging out-of-tree MT7927 WiFi 7 + Bluetooth 5.4 patches to Arch Linux,
Debian, and Fedora users while upstream review is in progress. Supports 10+
hardware variants with firmware extraction tooling and an 8-hour stability test
suite.

**[jig](https://github.com/jetm/jig)** - Go TUI for git workflows built with
Bubble Tea. Consolidates interactive staging, hunk-level add, diff viewing, commit
log browsing, fixup commits, and interactive rebase into one tool. Ships with
goreleaser, golangci-lint, and a 90% test coverage threshold.

**[glpkg](https://github.com/jetm/glpkg)** - Python CLI for the GitLab Generic
Package Registry, published on PyPI as `glpkg-cli`. Shell completion, GitHub
Actions CI/CD, and a comprehensive test suite.

## Technical Writing

**Linaro Engineering Blog**
- [From Replace It with Intel to Upstream: Bringing MediaTek Bluetooth/WiFi 7 to Linux](https://www.linaro.org/blog/from-replace-it-with-intel-to-upstream-bringing-mediatek-bluetooth-wifi-7-to-linux/) - Mar 2026.
  MediaTek's MT7927 WiFi 7 chip shipped on 15+ flagship products from ASUS,
  Gigabyte, Lenovo, MSI, and TP-Link with zero Linux support. Through community
  reverse-engineering and 20 upstream kernel patches, full WiFi 7 (2+ Gbps on
  6 GHz) and Bluetooth 5.4 now work on every major Linux distribution. Once
  merged, every distribution gets support automatically - eliminating the
  maintenance burden of out-of-tree drivers and reducing hardware support risk
  for silicon vendors and OEMs alike.
- [5 Common Mistakes When Enabling Your Arm Device in the Automated Test Lab](https://www.linaro.org/blog/5-common-mistakes-when-enabling-your-arm-device-in-the-automated-test-lab/) - Mar 2026.
  Bringing up a new hardware platform in an automated test environment is rarely
  simple. For many engineers working with Arm development boards, the first
  experience with automated testing infrastructure quickly turns into a debugging
  session - silent serial consoles, misconfigured USB ports, unexpected voltage
  levels, and devices that refuse to boot. A real-world debugging journey while
  enabling a new device on a Linaro Automation Appliance (LAA), used by ONELab,
  Linaro's automated testing service for Arm platforms.
- [Enabling Standard Linux on Arm: How ONELab Accelerates Interoperability and SystemReady Continuous Compliance](https://www.linaro.org/blog/enabling-standard-linux-on-arm-how-onelab-accelerates-interoperability-and-systemready-continuous-compliance/) - Mar 2026.
  Most Arm embedded platforms still struggle to boot standard Linux distributions
  due to fragmented firmware implementations and non-standard boot flows. ONELab
  by Linaro addresses this by providing fully automated testing infrastructure
  that validates firmware compliance and OS interoperability on real hardware,
  aligned with Arm SystemReady requirements. Reduces what traditionally required
  12-24 weeks of manual validation to an automated 6-12 week compliance cycle.

**Personal Blog** - [jetm.github.io/blog](https://jetm.github.io/blog)

In-depth articles on Linux kernel driver development, Yocto, UEFI, and systems
engineering. Series covering the 15-month MT7927 upstream journey (4 posts), Intel
IPU6 mainline migration, DKMS packaging, and UEFI/QEMU workflows. Articles average
1,000–4,500 words with working code and patch series.
