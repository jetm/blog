---
title: "About"
description: "Systems engineer specializing in upstream Linux kernel development, embedded Linux, and CI/CD infrastructure. Available for consulting."
ShowToc: false
ShowReadingTime: false
ShowWordCount: false
outputs:
  - HTML
---

## About Me

I'm Javier Tia, a systems engineer specializing in upstream Linux kernel
development, embedded Linux, and CI/CD infrastructure for hardware platforms.

I work close to the metal - writing and upstreaming kernel drivers, building
Yocto-based BSPs from scratch, implementing UEFI Secure Boot on ARM platforms,
and building the automated test infrastructure that keeps it all green across
kernel releases.

Most recently I was a Senior Software Engineer at Linaro, where I worked on
upstream kernel enablement for MediaTek silicon, UEFI Secure Boot for ARM
platforms in the Trusted Substrate project, SOAFEE automotive Linux testing,
and LAVA-based CI/CD infrastructure for Yocto image validation.

My current active upstream work includes:

- **Linux kernel (linux-wireless@):** 18-patch series adding WiFi 7 support for
  the MediaTek MT7927/MT6639 (Filogic 380) to the mt76/mt7925 driver - v4,
  community-tested across 10+ hardware platforms
- **Linux kernel (linux-bluetooth@):** 8-patch series enabling MT6639 Bluetooth
  in btusb/btmtk - merged into bluetooth-next, targeting mainline Linux 7.1/7.2
- **libcamera (libcamera-devel@):** 3-patch AGC and AWB fix series for the Simple
  pipeline - merged May 2026. Follow-up 2-patch series (sumShift cleanup + OV2740
  tuning file) in review

Everything is public and verifiable via lore.kernel.org and the projects page.

## What I Work On

**Upstream kernel driver enablement**
Taking hardware from out-of-tree or vendor-supplied drivers to mainline Linux.
Full cycle: driver bring-up, patch series preparation, mailing list submission,
review iteration, and post-merge maintenance.

**Embedded Linux & Yocto BSP**
Board support package development and hardening on Yocto/OpenEmbedded. UEFI
Secure Boot implementation (U-Boot key enrollment, systemd-boot and kernel image
signing, OE-QA runtime test cases). OP-TEE integration. ARM platform bring-up.

**Embedded CI/CD infrastructure**
End-to-end CI/CD pipelines for embedded Linux: LAVA test lab setup, SQUAD result
aggregation, TuxSuite integration, GitLab CI pipelines for Yocto builds and
automated hardware testing. Turning manual QA into reproducible, automated
regression detection.

**Developer tooling**
Go and Python tooling for engineering workflows - CLI tools, GitLab/GitHub
automation, package registry tooling, and build system integration.

## About This Blog

This blog covers upstream kernel development, hardware bring-up, Yocto, and
systems engineering - written from real problems encountered in production. Posts
tend to be long, hands-on, and include the parts that didn't work the first time.

Published series: the full MT7927 story across 4 posts (WiFi 7 driver development
and BT upstreaming), Intel IPU6 webcam mainline migration, Yocto build systems
and CRA compliance, and practical Linux tooling.

## Work With Me

I'm currently available for consulting engagements. I work with:

- **Semiconductor vendors and OEMs** needing upstream kernel enablement for new
  silicon - WiFi, Bluetooth, camera, and platform drivers
- **Embedded product companies** building on Yocto/OpenEmbedded who need BSP
  bring-up, Secure Boot implementation, or OP-TEE integration
- **Engineering teams** building or improving automated test infrastructure for
  embedded Linux - LAVA lab setup, GitLab CI pipelines, hardware-in-the-loop
  testing

Engagements range from short advisory and code review to multi-month retainers.
Based in Costa Rica, working remotely with clients worldwide.

**Get in touch:**

- Email: [javier@jetm.me](mailto:javier@jetm.me)
- LinkedIn: [linkedin.com/in/javiertia](https://www.linkedin.com/in/javiertia/)
- GitHub: [github.com/jetm](https://github.com/jetm/)
