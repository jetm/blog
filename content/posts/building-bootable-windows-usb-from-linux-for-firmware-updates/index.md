---
title: "Building a Bootable Windows USB from Linux for Firmware Updates"
date: 2026-02-27T18:00:00Z
draft: false
description: "How to build a bootable Windows 11 USB drive entirely from Linux using QEMU, UEFI Secure Boot, and qemu-img - for when your hardware vendor only ships Windows firmware tools."
ShowToc: true
ShowReadingTime: true
tags:
  - "linux"
  - "firmware"
  - "qemu"
  - "windows"
  - "hardware"
  - "arch-linux"
categories:
  - "linux"
  - "hardware"
outputs:
  - "HTML"
---

Three devices on my PC have firmware that can only be updated through Windows tools: an ASMedia ASM4242 USB4 controller (ASUS firmware utility), an NZXT Kraken Elite AIO cooler (NZXT CAM), and a Razer Kiyo Pro Ultra webcam (Razer Synapse). Every other component - NVMe SSD, motherboard BIOS, fwupd-supported devices - has a Linux-native update path. These three don't, and their vendors show no interest in changing that.

The obvious answer is "just boot Windows." But I don't have a Windows partition, don't want one, and installing Windows to flash three firmware blobs is absurd. I needed a way to boot a fully configured Windows environment from Linux, run the vendor tools, and shut down. No permanent installation, no dual-boot, no repartitioning.

The solution: build a Windows 11 VM in QEMU, install the firmware tools inside it, and write the disk image directly to a USB drive. The entire pipeline runs on Linux. The image is reusable - update the tools, re-write, done.

This post walks through the end-to-end process, including the gotchas that aren't documented anywhere else.

## Why the Obvious Approaches Don't Work

There are several ways to get a Windows environment from Linux. Most of them don't work for this use case:

| Approach | Full Windows | GUI Apps Work | Created from Linux | Reusable |
|----------|-------------|---------------|-------------------|----------|
| WoeUSB | Requires install | After install | Yes | No |
| Ventoy (ISO) | Requires install | After install | Yes | No |
| WinPE (mkwinpeimg) | Minimal only | No (missing runtimes) | Yes | Yes |
| Hiren's Boot PE | Fixed toolset | No (can't add apps) | Partially | Yes |
| **Raw USB (qemu-img)** | **Boots directly** | **Yes** | **Yes** | **Yes** |

WoeUSB and bare ISOs require installing Windows to a USB drive every time - you're sitting through the OOBE, installing tools, and throwing it away. WinPE doesn't have the .NET runtimes and UI frameworks that NZXT CAM and Razer Synapse need. Hiren's Boot PE has a fixed set of tools that can't be extended.

A raw USB image wins on every axis. You build Windows once in a VM, write the disk image directly to a USB drive with `qemu-img convert -O raw`, and the USB boots natively - no bootloader, no intermediate format. The qcow2 already contains a full GPT layout (EFI partition, MSR, Windows NTFS), so the USB is indistinguishable from a Windows install. The firmware tools are pre-installed and ready. When you need to update, you boot the VM again, update the tools, and re-write the image.

## Building the VM

The VM needs UEFI Secure Boot and a TPM 2.0 device - Windows 11 refuses to install without both. QEMU provides these through OVMF (UEFI firmware) and swtpm (software TPM emulator).

The full script is at [jetm/dotfiles](https://github.com/jetm/dotfiles/tree/master/tooling/win11-usb) (`win11-fw-vm.sh`). Here's what it does:

**Prerequisites:** `qemu-desktop`, `edk2-ovmf`, `swtpm`, `mtools` (for `mkfs.fat` and `mcopy`), a [Windows 11 ISO](https://www.microsoft.com/software-download/windows11) (no license required - Windows runs unactivated indefinitely with minor cosmetic restrictions), and the [VirtIO drivers ISO](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso) (on Arch Linux, install `virtio-win` from the AUR instead).

**Disk creation.** A 40GB qcow2 image. This is the virtual disk that gets written to USB later. 40GB is enough for Windows 11 Pro plus a few firmware utilities.

```bash
qemu-img create -f qcow2 win11-fw.qcow2 40G
```

**OVMF VARS copy.** OVMF uses two files: `OVMF_CODE.secboot.4m.fd` (read-only firmware) and `OVMF_VARS.4m.fd` (writable NVRAM for boot variables, Secure Boot keys, etc.). The script copies the VARS file to a writable location so UEFI can persist changes across reboots:

```bash
cp /usr/share/edk2/x64/OVMF_VARS.4m.fd win11-fw_VARS.fd
```

**Floppy image with autounattend.xml.** Windows Setup scans removable media for `autounattend.xml` during installation. The script creates a FAT-formatted floppy image and copies the answer file into it using `mtools`:

```bash
dd if=/dev/zero of=autounattend.img bs=1440K count=1 status=none
mkfs.fat autounattend.img
mcopy -i autounattend.img autounattend.xml ::autounattend.xml
```

**swtpm daemon.** The software TPM runs as a background process, listening on a Unix socket:

```bash
swtpm socket --tpmstate dir=/tmp/swtpm-win11 \
    --ctrl type=unixio,path=/tmp/swtpm-win11/sock \
    --tpm2 --daemon
```

**QEMU launch.** The key flags:

```bash
qemu-system-x86_64 -enable-kvm -m 8G -cpu host -smp 4 \
    -machine q35,smm=on \
    -global driver=cfi.pflash01,property=secure,value=on \
    -drive if=pflash,format=raw,unit=0,file="$OVMF_CODE",readonly=on \
    -drive if=pflash,format=raw,unit=1,file="$VARS" \
    -chardev socket,id=chrtpm,path=/tmp/swtpm-win11/sock \
    -tpmdev emulator,id=tpm0,chardev=chrtpm \
    -device tpm-tis,tpmdev=tpm0 \
    -drive file=win11-fw.qcow2,format=qcow2,if=virtio \
    -device ahci,id=ahci \
    -device ide-cd,bus=ahci.0,drive=cd0,bootindex=0 \
    -drive id=cd0,if=none,format=raw,media=cdrom,file="$WIN_ISO",readonly=on \
    -device ide-cd,bus=ahci.1,drive=cd1 \
    -drive id=cd1,if=none,format=raw,media=cdrom,file="$VIRTIO_ISO",readonly=on \
    -drive file=autounattend.img,format=raw,if=floppy \
    -device usb-ehci -device usb-tablet \
    -device virtio-net-pci,netdev=net0 -netdev user,id=net0 \
    -display gtk
```

The important pieces: `q35,smm=on` with `secure=on` enables Secure Boot. The pflash drives load OVMF firmware and writable NVRAM. The TPM connects via the swtpm socket. The disk is VirtIO (faster than IDE/AHCI emulation, but needs drivers during install - hence the VirtIO ISO on the second CD drive). The floppy carries `autounattend.xml`. Network is user-mode NAT for downloading tools inside the VM.

## The Unattended Install

The `autounattend.xml` handles the entire Windows installation without user interaction. It runs across three passes:

**windowsPE pass** - the earliest phase, before Windows is installed:
- Loads VirtIO storage drivers (`vioscsi`, `viostor`) and network driver (`NetKVM`) from the second CD (`E:\`). Without these, Windows can't see the VirtIO disk or get network access.
- Partitions the disk as GPT: 260MB EFI System Partition (FAT32), 16MB MSR, and the remaining space as NTFS for Windows.
- Selects image index 6 from the ISO, which is Windows 11 Pro.

**specialize pass** - after file copy, before first boot:
- Sets the computer name to `FW-UPDATE`.
- Injects the BypassNRO registry hack (`HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\OOBE\BypassNRO = 1`). This is what lets you create a local account instead of signing in with a Microsoft account - Windows 11 normally forces online setup.

**oobeSystem pass** - the first-boot experience:
- Skips EULA, wireless setup, and privacy screens.
- Creates a local admin account named `User` with no password and auto-logon.

One quirk: on the very first boot, OVMF may show a "no bootable device" screen because the EFI boot variables haven't been written yet. Press Enter to reach the boot manager, select the CD, and the unattended install takes over from there. Subsequent boots go straight to Windows.

After Windows is running, install whatever firmware tools you need inside the VM. The VirtIO network gives you internet access for downloading installers. Then shut down Windows cleanly through the Start menu.

## Writing to USB

With the VM shut down, write the qcow2 disk directly to the USB drive as a raw image:

```bash
qemu-img convert -p -O raw win11-fw.qcow2 /dev/sdX
```

This converts the qcow2 (a sparse, copy-on-write format) to raw - a byte-for-byte disk image - and writes it directly to the USB device. The qcow2 already contains a full GPT layout from the Windows install (EFI System Partition, MSR, Windows NTFS partition), so the USB boots natively like a Windows install. No bootloader, no intermediate format, no VHD.

The `-p` flag shows a progress bar, which is useful since the write takes a few minutes.

You can verify the result with `qemu-img info`:

```text
$ qemu-img info win11-fw.qcow2
file format: qcow2
virtual size: 40 GiB (42949672960 bytes)
disk size: 12.4 GiB
```

The trade-off is write time: `qemu-img convert -O raw` writes the full 40 GiB regardless of how much space Windows actually uses inside the image. This takes a few minutes on a USB 3.x drive. The same trade-off existed with the fixed VHD approach, so nothing changes in practice.

## Boot and Flash

Reboot, hit F8 (ASUS boot menu), and select the USB drive. Windows boots directly to the desktop with the firmware tools pre-installed.

Connect the devices that need updating (the USB4 controller is on the motherboard, so it's always connected; the NZXT AIO and Razer webcam go over USB), run each vendor's tool, flash, and shut down. Remove the USB, boot back into Linux.

### Re-use

When firmware tools release new versions or you need to add another tool, run the VM script again - it reuses the existing qcow2 disk:

```bash
win11-fw-vm.sh
```

Update tools inside the VM, shut down, and re-write the image to USB with `qemu-img convert -O raw`.

## References

- [VirtIO drivers ISO](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso) - Red Hat VirtIO drivers for Windows guests
- [OVMF (edk2)](https://github.com/tianocore/edk2) - UEFI firmware for virtual machines
- [swtpm](https://github.com/stefanberger/swtpm) - Software TPM 2.0 emulator
- [Microsoft unattended reference](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/automate-windows-setup) - autounattend.xml documentation
- [Full scripts on GitHub (jetm/dotfiles)](https://github.com/jetm/dotfiles/tree/master/tooling/win11-usb) - `win11-fw-vm.sh` and `autounattend.xml`
