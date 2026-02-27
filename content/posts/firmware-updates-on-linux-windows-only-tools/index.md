---
title: "Firmware Updates on Linux When the Tools Are Windows-Only"
date: 2026-02-27T18:00:00Z
draft: true
description: "How to build a bootable Windows 11 VHD entirely from Linux using QEMU, UEFI Secure Boot, and Ventoy - for when your hardware vendor only ships Windows firmware tools."
ShowToc: true
ShowReadingTime: true
tags:
  - "linux"
  - "firmware"
  - "qemu"
  - "ventoy"
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

The solution: build a Windows 11 VM in QEMU, install the firmware tools inside it, convert the disk to a VHD, and boot it directly from a Ventoy USB. The entire pipeline runs on Linux. The VHD is reusable - update the tools, re-export, done.

This post walks through the end-to-end process, including the gotchas that aren't documented anywhere else.

## Why Not Just...

There are several ways to get a Windows environment from Linux. Most of them don't work for this use case:

| Approach | Full Windows | GUI Apps Work | Created from Linux | Reusable |
|----------|-------------|---------------|-------------------|----------|
| WoeUSB | Requires install | After install | Yes | No |
| Ventoy (ISO) | Requires install | After install | Yes | No |
| WinPE (mkwinpeimg) | Minimal only | No (missing runtimes) | Yes | Yes |
| Hiren's Boot PE | Fixed toolset | No (can't add apps) | Partially | Yes |
| **Ventoy (VHD)** | **Boots directly** | **Yes** | **Yes** | **Yes** |

WoeUSB and bare ISOs require installing Windows to a USB drive every time - you're sitting through the OOBE, installing tools, and throwing it away. WinPE doesn't have the .NET runtimes and UI frameworks that NZXT CAM and Razer Synapse need. Hiren's Boot PE has a fixed set of tools that can't be extended.

A Ventoy VHD wins on every axis. You build Windows once in a VM, export the disk as a VHD, and Ventoy boots it directly from USB as if it were installed on bare metal. The firmware tools are pre-installed and ready. When you need to update, you boot the VM again, update the tools, re-export, and re-copy.

## Building the VM

The VM needs UEFI Secure Boot and a TPM 2.0 device - Windows 11 refuses to install without both. QEMU provides these through OVMF (UEFI firmware) and swtpm (software TPM emulator).

The full script is at [jetm/dotfiles](https://github.com/jetm/dotfiles) (`win11-fw-vm.sh`). Here's what it does:

**Prerequisites:** `qemu-desktop`, `edk2-ovmf`, `swtpm`, `mtools` (for `mkfs.fat` and `mcopy`), a Windows 11 ISO, and the [VirtIO drivers ISO](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso).

**Disk creation.** A 40GB qcow2 image. This is the virtual disk that becomes the VHD later. 40GB is enough for Windows 11 Pro plus a few firmware utilities.

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

## Converting to VHD

With the VM shut down, convert the qcow2 disk to VHD format:

```bash
qemu-img convert -O vpc -o subformat=fixed win11-fw.qcow2 win11-fw.vhd
```

### The 0x12F gotcha

The `-o subformat=fixed` flag is critical. Without it, `qemu-img` creates a **dynamic** VHD - and dynamic VHDs will not boot.

When Windows does native VHD boot, the early boot environment (bootmgr, winload) needs to map the VHD's contents into physical memory at predictable offsets. A fixed VHD is a 1:1 image of the virtual disk plus a 512-byte footer - the bootloader can treat it as a raw disk. A dynamic VHD uses a Block Allocation Table (BAT) where blocks are allocated on demand and stored out-of-order. The Windows boot environment can't parse the BAT because the drivers for it haven't loaded yet.

The result is a blue screen with `VHD_BOOT_INITIALIZATION_FAILED` (error code 0x12F). No diagnostic information, no hint that VHD format is the problem.

You can verify the format with `qemu-img info`:

```text
# Dynamic VHD (BAD - will 0x12F)
$ qemu-img info win11-fw-dynamic.vhd
file format: vpc
virtual size: 40 GiB
disk size: 12.4 GiB           # <-- smaller than virtual size = dynamic

# Fixed VHD (GOOD)
$ qemu-img info win11-fw.vhd
file format: vpc
virtual size: 40 GiB
disk size: 40 GiB             # <-- matches virtual size = fixed
```

The trade-off is size: a fixed VHD is the full 40GB regardless of how much space Windows actually uses. This is fine for a USB drive; it would be a problem if you were short on storage.

## Ventoy USB Setup

[Ventoy](https://www.ventoy.net/) is an open-source tool that creates a bootable USB where you just drop ISO/VHD/WIM files and boot them directly. Install it on a USB drive:

```bash
sudo /opt/ventoy/Ventoy2Disk.sh -i /dev/sdX   # DESTROYS ALL DATA ON USB
```

### The mkexfatfs symlink

On Arch, Ventoy's install script looks for `mkexfatfs` but the package (`exfatprogs`) provides `mkfs.exfat`. If the install fails with a missing command error:

```bash
sudo ln -s /usr/bin/mkfs.exfat /usr/local/bin/mkexfatfs
```

### The vhdboot helper

Without an extra helper file, Ventoy won't boot VHDs correctly in UEFI mode. Instead of launching Windows, it presents a file picker showing every EFI file on the USB.

The fix is `ventoy_vhdboot.img` - a small helper image that Ventoy loads before the VHD to set up the EFI boot chain. Download it from the [ventoy/vhdiso](https://github.com/ventoy/vhdiso/releases) GitHub releases:

```bash
curl -sL -o /tmp/ventoy_vhdboot.zip \
  https://github.com/ventoy/vhdiso/releases/download/v3.0/ventoy_vhdboot.zip
unzip -o /tmp/ventoy_vhdboot.zip -d /tmp
```

The ZIP contains two versions: `Win10Based` and `Win11Based`. Use `Win10Based` - it's compatible with Windows 11 and is more widely tested.

### The scanning problem

Ventoy scans the entire USB partition at boot to find bootable images. On a large drive with other data, this takes forever. My 950GB USB drive (which doubles as general storage) took over 8 minutes to scan because Ventoy was walking 762GB of files looking for ISOs and VHDs.

The fix is to set Ventoy's image list mode to `whitelist`, so it only looks for the files you specify.

### Putting it all together

Mount the Ventoy USB partition and set up the config:

```bash
sudo mount /dev/sdX1 /mnt
sudo mkdir -p /mnt/ventoy

# Install vhdboot helper (Win10Based version, compatible with Windows 11)
sudo cp /tmp/ventoy_vhdboot/Win10Based/ventoy_vhdboot.img /mnt/ventoy/
```

Create `/mnt/ventoy/ventoy.json`:

```json
{
    "control_uefi": [
        { "VTOY_SECONDARY_BOOT_MENU": "0" },
        { "VTOY_FILE_FLT_EFI": "1" }
    ],
    "auto_install": [
        {
            "image": "/win11-fw.vhd",
            "timout": 0
        }
    ],
    "image_list_mode": "whitelist",
    "image_list": [
        "/win11-fw.vhd"
    ]
}
```

Three fixes in one config:
- `VTOY_SECONDARY_BOOT_MENU: 0` - disables the "Boot in Wimboot mode?" prompt.
- `VTOY_FILE_FLT_EFI: 1` - filters stray EFI files that confuse the boot menu.
- `image_list_mode: whitelist` with `image_list` - only scans for `/win11-fw.vhd`, skipping the full drive walk.

Copy the VHD and unmount:

```bash
sudo cp win11-fw.vhd /mnt/
sync
sudo umount /mnt
```

## Boot and Flash

Reboot, hit F8 (ASUS boot menu), select the Ventoy USB. Ventoy shows one entry: `win11-fw.vhd`. Select it, Windows boots to the desktop with the firmware tools pre-installed.

Connect the devices that need updating (the USB4 controller is on the motherboard, so it's always connected; the NZXT AIO and Razer webcam go over USB), run each vendor's tool, flash, and shut down. Remove the USB, boot back into Linux.

### Re-use

When firmware tools release new versions or you need to add another tool, run the VM script again - it reuses the existing qcow2 disk:

```bash
win11-fw-vm.sh
```

Update tools inside the VM, shut down, re-convert to VHD, re-copy to USB. The Ventoy config doesn't change.

## References

- [Ventoy](https://www.ventoy.net/) - Open-source tool for creating bootable USB drives
- [ventoy/vhdiso releases](https://github.com/ventoy/vhdiso/releases) - VHD boot helper images
- [VirtIO drivers ISO](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso) - Red Hat VirtIO drivers for Windows guests
- [OVMF (edk2)](https://github.com/tianocore/edk2) - UEFI firmware for virtual machines
- [swtpm](https://github.com/stefanberger/swtpm) - Software TPM 2.0 emulator
- [Microsoft unattended reference](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/automate-windows-setup) - autounattend.xml documentation
- [Full scripts on GitHub (jetm/dotfiles)](https://github.com/jetm/dotfiles) - `win11-fw-vm.sh` and `autounattend.xml`
