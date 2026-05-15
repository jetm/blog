---
title: "Javier Tia - CV"
description: "Ingeniero de Sistemas Senior especializado en desarrollo upstream del kernel Linux, Linux embebido, UEFI Secure Boot e infraestructura CI/CD."
ShowToc: false
ShowReadingTime: false
ShowWordCount: false
ShowBreadCrumbs: false
ShowPostNavLinks: false
layout: "cv-es"
outputs:
  - HTML
---

## Perfil Profesional

Ingeniero de Sistemas Senior con más de 15 años de experiencia en software
embebido en producción, especializado en desarrollo upstream del kernel Linux,
Yocto/OpenEmbedded, seguridad embebida (UEFI Secure Boot, OP-TEE) e
infraestructura CI/CD para hardware. Colaborador activo upstream con fusiones
recientes en Linux mainline (Bluetooth, libcamera) y series de parches en
revisión activa en linux-wireless@, linux-firmware y Yocto. Disponible para
contratos de consultoría, posiciones remotas a tiempo completo y abierto a
reubicación.

## Habilidades Técnicas

**Kernel y firmware:** Desarrollo del kernel Linux, DKMS, U-Boot, UEFI/Secure Boot,
OP-TEE, drivers de dispositivo (WiFi/mac80211/cfg80211, Bluetooth/btusb/btmtk, PCIe,
cámara/libcamera)

**Sistemas de build:** Yocto/OpenEmbedded, BitBake, kas, (meta-arm, meta-openembedded,
meta-secure-core), Make, CMake, Meson, Autotools, repo tool

**CI/CD y testing:** LAVA, SQUAD, TuxSuite, GitLab CI, GitHub Actions

**Lenguajes:** C, C++, Go, Python, Shell, Perl

**Seguridad:** UEFI Secure Boot, measured boot, TPM, sbsign, gestión de variables EFI
(libefivar)

**Backend:** Django, REST APIs, automatización con shell

**Wireless:** mac80211, cfg80211, familia de drivers mt76, pila Bluetooth btusb/btmtk

**Plataformas:** ARM/AArch64/ARMv8, x86-64, RISC-V, QEMU, Raspberry Pi, NXP i.MX,
Qualcomm, MediaTek, Xilinx/AMD KV260, Synquacer

**Embebido:** Desarrollo de BSP, bring-up de placa, device tree (DTS/DTB),
compilación cruzada, UART/I2C/SPI, systemd, udev, initramfs

## Experiencia

### Ingeniero Open Source (Independiente)
**Mar 2026 – Actualidad | Remoto**

Tras dejar Linaro, continúo aportando parches upstream al kernel Linux,
libcamera y Yocto/OE. Mantengo herramientas open source y publico
artículos técnicos.

- Desarrollé shipcheck, un auditor de cumplimiento del Cyber Resilience Act
  (CRA) europeo para builds de Linux embebido y Yocto. Lee lo que produce
  un build de Yocto - SBOMs, salida de escaneo CVE, artefactos de firma y
  manifiestos de licencias - y genera una puntuación de preparación y un
  expediente CRA multi-fichero. Publicado en PyPI.
- Mantuve mediatek-mt7927-dkms siguiendo la serie upstream de parches para
  WiFi 7.
- Desarrollé una serie de parches de seguimiento para libcamera con el
  archivo de calibración del sensor OV2740 extraído del binario AIQB de
  Intel.
- Publiqué artículos técnicos sobre rendimiento de builds Yocto,
  cumplimiento CRA, calibración de sensores libcamera, y la experiencia de
  afrontar una búsqueda de empleo como ingeniero senior.

---

### Linaro – Ingeniero de Sistemas Senior
**Sep 2022 – Mar 2026 | Remoto**

Linaro es una organización de ingeniería colaborativa de código abierto centrada en
hardware y software basados en Arm. Trabajé en habilitación upstream del kernel,
seguridad embebida, infraestructura CI/CD y bring-up de plataforma para los stacks
de referencia ARM utilizados por las empresas miembro de Linaro.

**Kernel y Firmware Upstream**
- Desarrollé una serie de 18 parches para WiFi 7 del MediaTek MT7927/MT6639
  (Filogic 380) en el driver mt76/mt7925 (linux-wireless@, v4); probado en
  comunidad en más de 10 plataformas de hardware con 9 etiquetas Tested-by de
  ASUS, Lenovo, Foxconn y AMD
- Desarrollé una serie de 8 parches para Bluetooth habilitando el MT6639 en
  btusb/btmtk; fusionado en mainline (abril 2026); firmware companion enviado
  a linux-firmware
- Desarrollé una serie de 3 parches para libcamera corrigiendo AGC/AWB en el
  pipeline Simple para sensores con salida >8 bits (libcamera-devel@);
  fusionado en mayo de 2026; serie de seguimiento de 2 parches (OV2740) en
  revisión
- Mantuve ramas estables del kernel Linux para targets embebidos ARM (KV260,
  RockPi4, Synquacer, AVA, TI SK-AM62P-LP, NXP i.MX 8M, Raspberry Pi,
  genericarm64): backports de CVE, parches de estabilidad y cadencia regular
  de releases
- Contribuí parches upstream a u-boot/u-boot para bring-up de plataformas ARM,
  configuración de la cadena Secure Boot y gestión de claves UEFI

**Seguridad Embebida**
- Diseñé e integré upstream en meta-arm una implementación completa de UEFI
  Secure Boot: enrolamiento de claves en U-Boot, firma de kernel con
  systemd-boot, clase BitBake sbsign, casos de test OE-QA en runtime y
  GitLab CI; fusionado tras 8 ciclos de revisión (oct. 2024, revisado por
  Jon Mason, ARM)
- Desarrollé parches para OP-TEE/optee-client: activación de tee-supplicant
  basada en udev sustituyendo dependencias estáticas, resolviendo fallos en
  plataformas con múltiples /dev/teepriv* (fusionado 2023)
- Contribuí a la capa de seguridad Yocto meta-ledge-secure: integración de
  Secure Boot, cifrado de disco y gestión de variables EFI

**Infraestructura CI/CD**
- Desarrollé GPIT: CI/CD end-to-end en GitLab para imágenes Yocto Trusted
  Substrate con testing automatizado en hardware LAVA, análisis de resultados
  OEQA, análisis de logs en dos pasadas y reporte por correo en targets ARM
  (KV260, RockPi4, Synquacer, AVA, genericarm64); 19 MRs fusionados
- Construí y mantuve CI/CD basado en LAVA para testing de Linux embebido en
  hardware ARM (Synquacer, RockPi4, KV260, AVA): planes de test, pipelines de
  capsule update, scripts de merge de imágenes e integración con GitLab CI;
  más de 40 MRs fusionados
- Integré placas de desarrollo ARM en pipelines de LAVA, SQUAD y TuxSuite,
  aumentando la cobertura de tests automáticos en hardware y reduciendo los
  ciclos de validación manual
- Desarrollé el framework de testing SOAFEE: motor de contenedores, k3s,
  virtualización Xen, OpenAD Kit y testing de cumplimiento Linux ABI con
  reporting TAP e integración LAVA; más de 30 MRs fusionados

**Yocto/OpenEmbedded**
- Responsable del sistema de build Yocto/OE que producía todo el software y
  kernels para los productos Linux embebido de referencia de ARM (Trusted
  Substrate y SOAFEE) en KV260, RockPi4, Synquacer, AVA, TI SK-AM62P-LP,
  NXP i.MX 8M, Raspberry Pi y genericarm64; gestionando capas de
  meta-openembedded, meta-arm, meta-secure-core y meta-arm-bsp
- Establecí configuraciones BitBake reproducibles y estrategias de sstate-cache,
  reduciendo fallos de build en el equipo
- Revisor activo y contribuidor de parches en meta-openembedded, meta-arm y
  meta-secure-core

**Backend y Herramientas**
- Contribuí funcionalidades a la REST API Django de ONELab (plataforma de
  testing de interoperabilidad IoT/edge de Linaro): pipelines de despliegue
  automatizado en GitLab CI y herramientas de shell, reduciendo el esfuerzo
  de incorporación manual de dispositivos
- Integré herramientas de variables EFI (rhboot/efivar) en stacks de firmware
  embebido, habilitando el acceso a variables EFI en runtime desde el espacio
  de usuario Linux en plataformas ARM

---

### Hewlett Packard Enterprise / Aruba – Ingeniero de Sistemas
**Mar 2011 – Sep 2022 | Remoto**

11 años de ingeniería de software embebido en producción en switches de red
HPE/Aruba con Linux embebido. Trabajé en todo el stack de firmware para hardware
de red empresarial de alta disponibilidad.

- Participé en un esfuerzo a gran escala de rearquitectura para modularizar y
  modernizar el firmware de switches HPE/Aruba en producción con Linux embebido
  en una flota de dispositivos de red empresariales
- Desarrollé y mantuve BSPs de Linux embebido para plataformas de switches de
  red: configuración del kernel, integración de drivers, bring-up de placa y
  testing a nivel de sistema
- Trabajé en sistemas de build de firmware, integración CI y release engineering
  para software de switches embebidos
- Contribuí a esfuerzos multidisciplinares abarcando kernel, espacio de usuario
  y componentes del plano de gestión en una organización de ingeniería grande y
  distribuida

## Contribuciones Upstream

Parches upstream seleccionados y más de 200 PRs/MRs fusionados: [jetm.github.io/blog/projects](https://jetm.github.io/blog/projects/)

## Proyectos y Herramientas

- [mediatek-mt7927-dkms](https://github.com/jetm/mediatek-mt7927-dkms)
- [jig](https://github.com/jetm/jig)
- [glpkg](https://github.com/jetm/glpkg)

Descripciones completas: [jetm.github.io/blog/projects](https://jetm.github.io/blog/projects/)

## Artículos Técnicos

**Blog de Ingeniería de Linaro**
- [From Replace It with Intel to Upstream: Bringing MediaTek Bluetooth/WiFi 7 to Linux](https://www.linaro.org/blog/from-replace-it-with-intel-to-upstream-bringing-mediatek-bluetooth-wifi-7-to-linux/) - Mar 2026
- [5 Common Mistakes When Enabling Your Arm Device in the Automated Test Lab](https://www.linaro.org/blog/5-common-mistakes-when-enabling-your-arm-device-in-the-automated-test-lab/) - Mar 2026
- [Enabling Standard Linux on Arm: How ONELab Accelerates Interoperability and SystemReady Continuous Compliance](https://www.linaro.org/blog/enabling-standard-linux-on-arm-how-onelab-accelerates-interoperability-and-systemready-continuous-compliance/) - Mar 2026

**Blog Personal** - [jetm.github.io/blog](https://jetm.github.io/blog)
