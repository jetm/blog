---
title: "What's Next: Available for New Opportunities"
date: 2026-04-01T12:00:00Z
draft: false
description: "After 3.5 years at Linaro I was laid off in March 2026. Here is what I have been working on and what I am looking for next."
ShowToc: false
ShowReadingTime: true
tags:
  - "career"
  - "open-source"
  - "linux"
  - "kernel"
  - "embedded"
  - "yocto"
outputs:
  - "HTML"
---

At the end of March 2026 I was laid off from Linaro as part of a round of
cuts. After 3.5 years working on upstream kernel enablement, embedded
security, and CI/CD infrastructure for ARM-based platforms, it came as a
surprise - but I am using the time well.

## What I have been doing since

The upstream work did not stop. I currently have three active patch series
under review:

- An 18-patch WiFi 7 series on **linux-wireless@** adding full support for
  the MediaTek MT7927 (Filogic 380) to the mt76/mt7925 driver. The series
  is at v4, community-tested across 10+ hardware platforms with 9
  Tested-by tags from ASUS, Lenovo, Foxconn, and AMD. Phoronix covered
  it: [MediaTek MT7927 WiFi 7 Linux Support Coming
  Together](https://www.phoronix.com/news/MediaTek-MT7927-WiFi-Linux).

- An 8-patch Bluetooth series on **linux-bluetooth@** enabling the MT6639
  companion chip in btusb/btmtk. These patches were just merged into
  **bluetooth-next**, which means they will land in the next kernel
  release.

- A 3-patch series on **libcamera-devel@** fixing AGC and AWB bugs in the
  Simple pipeline that were preventing a clean migration away from Intel's
  proprietary IPU6 driver stack on ThinkPads, XPS, and Surface laptops.
  Reviewed by engineers from Red Hat, Collabora, and Ideas on Board.

I also published three articles on the [Linaro Engineering
Blog](https://www.linaro.org/blog/) in March covering the MT7927 upstream
journey, LAA device integration, and ONELab compliance testing - and I
gave a talk pitch to the KernelCI TSC about lore-first patch testing and
KCIDB integration.

All of my open source work is publicly verifiable at
[jetm.github.io/blog/projects](/blog/projects/).

## What I am looking for

I am open to **full-time remote roles** and **consulting or freelance
engagements** in:

- Linux kernel development and upstream driver work
- Yocto/OpenEmbedded BSP development and layer maintenance
- Embedded security (UEFI Secure Boot, OP-TEE, measured boot)
- CI/CD infrastructure for embedded Linux (LAVA, SQUAD, TuxSuite, GitLab CI)

I work fully remote from Costa Rica and have done so throughout my career.
I am comfortable across time zones and open to relocation for the right
opportunity.

## Get in touch

If you are working on something that fits - or know someone who is - I
would love to hear from you.

- **Email:** [javier@jetm.me](mailto:javier@jetm.me)
- **LinkedIn:** [cr.linkedin.com/in/javiertia](https://cr.linkedin.com/in/javiertia)
- **GitHub:** [github.com/jetm](https://github.com/jetm)
- **CV:** [jetm.github.io/blog/cv](/blog/cv/)
