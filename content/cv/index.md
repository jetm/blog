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

Senior Systems Engineer with 15+ years of production embedded software
experience, specializing in upstream Linux kernel development,
Yocto/OpenEmbedded, embedded security (UEFI Secure Boot, OP-TEE), and CI/CD
infrastructure for hardware. Active upstream contributor with recent merges
into mainline Linux (Bluetooth, libcamera) and active patch series under
review on linux-wireless@, linux-firmware, and Yocto. Open to consulting
engagements and full-time remote roles.

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

### Open Source Developer (Independent)
**Jan 2014 – Present | Remote**

Upstreaming Linux kernel, libcamera, and Yocto/OE patches. Maintaining
open-source tooling work and writing technical articles.

- Built shipcheck tool, an EU Cyber Resilience Act (CRA) compliance auditor
  for embedded Linux and Yocto builds. It reads what a Yocto build emits -
  SBOMs, CVE scan output, signing artifacts, and license manifests - and
  produces a readiness score and multi-file CRA evidence dossier. Published
  on PyPI.
- Built bspctl, a Python CLI wrapping kas and kas-container for reproducible
  Yocto BSP builds. Adds pre-flight environment checks, run-time tuning
  without modifying build YAMLs, vendor manifest translation (NXP i.MX,
  TI Sitara, bitbake-setup workspaces), and structured per-run logs for
  failure triage. Available on PyPI.
- Maintained mediatek-mt7927-dkms and tracked the upstream WiFi 7 patch
  series.
- Authored a libcamera follow-up patch series for OV2740 sensor tuning
  calibrated from Intel's AIQB binary.
- Published technical writing on Yocto build performance, CRA compliance,
  libcamera sensor calibration, and the experience of navigating a job
  search as a senior engineer.
- Maintains Arch Linux (AUR) packages for upstream kernel drivers and
  developer tooling.

---

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
  mainline (April 2026); companion firmware submitted to linux-firmware
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

Selected upstream patches and 200+ merged PRs/MRs: [jetm.github.io/blog/projects](https://jetm.github.io/blog/projects/)

## Projects & Tooling

- [mediatek-mt7927-dkms](https://github.com/jetm/mediatek-mt7927-dkms)
- [jig](https://github.com/jetm/jig)
- [glpkg](https://github.com/jetm/glpkg)
- [bspctl](https://github.com/jetm/bspctl)

Full descriptions: [jetm.github.io/blog/projects](https://jetm.github.io/blog/projects/)

## Technical Writing

**Linaro Engineering Blog**
- [From Replace It with Intel to Upstream: Bringing MediaTek Bluetooth/WiFi 7 to Linux](https://www.linaro.org/blog/from-replace-it-with-intel-to-upstream-bringing-mediatek-bluetooth-wifi-7-to-linux/) - Mar 2026
- [5 Common Mistakes When Enabling Your Arm Device in the Automated Test Lab](https://www.linaro.org/blog/5-common-mistakes-when-enabling-your-arm-device-in-the-automated-test-lab/) - Mar 2026
- [Enabling Standard Linux on Arm: How ONELab Accelerates Interoperability and SystemReady Continuous Compliance](https://www.linaro.org/blog/enabling-standard-linux-on-arm-how-onelab-accelerates-interoperability-and-systemready-continuous-compliance/) - Mar 2026

**Personal Blog** - [jetm.github.io/blog](https://jetm.github.io/blog)
