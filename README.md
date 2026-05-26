# Windows 11 on Moto Edge+ 2022 UW (XT2201-4)

## Windows on ARM Porting Guide — WSL Edition

> ⚠️ **PIONEER TERRITORY** — No complete WoA port exists for `hiphi` as of mid-2026. This guide does everything possible inside WSL (Ubuntu), then hands off to native Windows only where WSL can’t reach (DISM, WinPE, BCD). Expect weeks of iteration, broken boots, and missing drivers.

-----

## Table of Contents

1. [Device Info](#device-info)
1. [WSL Setup](#wsl-setup)
1. [Prerequisites](#prerequisites)
1. [Stage 1 — Unlock & Prep](#stage-1--unlock--prep)
1. [Stage 2 — UEFI Port (EDK2 / Renegade)](#stage-2--uefi-port)
1. [Stage 3 — Partition Layout](#stage-3--partition-layout)
1. [Stage 4 — Deploy Windows ARM64](#stage-4--deploy-windows-arm64)
1. [Stage 5 — Driver Porting](#stage-5--driver-porting)
1. [Stage 6 — First Boot & OOBE](#stage-6--first-boot--oobe)
1. [Known Limitations](#known-limitations)
1. [Resources](#resources)

-----

## Device Info

|Field     |Value                                                         |
|----------|--------------------------------------------------------------|
|Codename  |`hiphi`                                                       |
|Variants  |XT2201-1 (global), XT2201-3 (Boost), **XT2201-4 (Verizon UW)**|
|SoC       |Snapdragon 8 Gen 1 (SM8450)                                   |
|RAM       |8GB LPDDR5                                                    |
|Storage   |512GB UFS 3.1                                                 |
|Display   |6.7” pOLED, 144Hz                                             |
|Bootloader|Motorola fastboot, Secure Boot enforced                       |

-----

## WSL Setup

### What WSL Can and Can’t Do

|Task                   |WSL? |Notes                        |
|-----------------------|-----|-----------------------------|
|Build EDK2 UEFI        |✅ Yes|Full Linux toolchain in WSL  |
|ADB / fastboot         |✅ Yes|Via USBIPD passthrough       |
|Partition the phone    |✅ Yes|ADB shell → sgdisk on device |
|Build ARM64 WinPE image|❌ No |Needs Windows ADK on host    |
|DISM image deployment  |❌ No |Needs Windows DISM on host   |
|BCD configuration      |❌ No |Needs Windows bcdedit on host|
|Extract .wim from ISO  |✅ Yes|`7z` or `wimlib` in WSL      |

**The split:** WSL handles all Linux build work and ADB. Switch to a normal PowerShell/CMD window for the Windows-native deployment steps (Stages 4–5).

-----

### Install WSL2 + Ubuntu

Open PowerShell as admin:

```powershell
wsl --install -d Ubuntu-24.04
wsl --set-default-version 2
```

Reboot when prompted, then finish Ubuntu first-time setup.

-----

### USB Passthrough via USBIPD

WSL2 can’t access USB devices by default. You need `usbipd-win` to pass the phone through.

**On Windows (PowerShell as admin):**

```powershell
winget install usbipd
```

**In WSL:**

```bash
sudo apt update && sudo apt install linux-tools-generic hwdata -y
sudo update-alternatives --install /usr/local/bin/usbip usbip \
  /usr/lib/linux-tools/$(ls /usr/lib/linux-tools)/usbip 20
```

**Each session — attach the phone:**

1. Plug phone in, put it in fastboot or ADB mode
1. In PowerShell (admin):

```powershell
usbipd list
# Find your phone — note the BUSID (e.g. 2-3)
usbipd bind --busid 2-3
usbipd attach --wsl --busid 2-3
```

1. In WSL, verify:

```bash
lsusb
# Should show Motorola / Android device
adb devices
```

> You’ll need to re-run `usbipd attach` every time you replug the phone or switch USB modes (ADB ↔ fastboot). The BUSID stays the same per port.

-----

## Prerequisites

### Hardware

- Moto Edge+ 2022 UW with **OEM unlocked bootloader**
- USB-C cable (USB 3.0+ recommended, direct to PC — no hubs for flashing)
- PC running Windows 10/11 with WSL2 installed

### Software — WSL side

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y \
  git build-essential uuid-dev iasl nasm python3 python3-distutils \
  android-tools-adb android-tools-fastboot \
  sgdisk parted dosfstools e2fsprogs \
  curl wget 7zip wimtools
```

### Software — Windows side (host)

- **Windows ADK** + **WinPE add-on** — [download from Microsoft](https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install)
- **UUP Dump script** to build ARM64 ISO — [uupdump.net](https://uupdump.net)

-----

## Stage 1 — Unlock & Prep

All commands run in WSL unless noted.

### 1.1 Attach Phone and Verify ADB

```bash
# Attach via usbipd first (see WSL Setup above)
adb devices
# Should show: <serial>    device
```

### 1.2 Unlock the Bootloader

The XT2201-4 (Verizon UW) requires a Motorola OEM unlock token. If you haven’t done this yet:

```bash
adb reboot bootloader
# Wait for fastboot mode

fastboot oem device-info
# Look for: "Device unlocking enabled" = true
# If false, flash RETUS firmware first — see Resources
```

Re-attach after reboot (usbipd detaches on device reboot):

```powershell
# PowerShell (admin)
usbipd attach --wsl --busid 2-3
```

```bash
fastboot oem unlock YOUR_UNLOCK_KEY
```

### 1.3 Flash TWRP

```bash
fastboot flash recovery twrp-hiphi.img
fastboot reboot recovery
```

Re-attach USB after it reboots into recovery.

### 1.4 Back Up Critical Partitions

**Do this now. These partitions are device-unique — losing them = soft brick.**

```bash
# ADB shell on device
adb shell "dd if=/dev/block/by-name/persist of=/sdcard/persist.img"
adb shell "dd if=/dev/block/by-name/modemst1 of=/sdcard/modemst1.img"
adb shell "dd if=/dev/block/by-name/modemst2 of=/sdcard/modemst2.img"
adb shell "dd if=/dev/block/by-name/fsg of=/sdcard/fsg.img"
adb shell "dd if=/dev/block/by-name/nvm of=/sdcard/nvm.img"

# Pull to WSL
mkdir -p ~/hiphi-backups
adb pull /sdcard/persist.img ~/hiphi-backups/
adb pull /sdcard/modemst1.img ~/hiphi-backups/
adb pull /sdcard/modemst2.img ~/hiphi-backups/
adb pull /sdcard/fsg.img ~/hiphi-backups/
adb pull /sdcard/nvm.img ~/hiphi-backups/
```

Copy the backup folder to Windows too:

```bash
cp -r ~/hiphi-backups/ /mnt/c/Users/$USER/hiphi-backups/
```

-----

## Stage 2 — UEFI Port

Build the Renegade EDK2 UEFI entirely in WSL.

### 2.1 Clone Renegade

```bash
cd ~
git clone https://github.com/edk2-porting/edk2-msm.git
cd edk2-msm
git submodule update --init --recursive
```

### 2.2 Check SM8450 Device Support

```bash
ls Platforms/SM8450/
```

If `hiphi` / `MotoEdgePlus2022` isn’t listed, create it from the closest SM8450 reference:

```bash
cp -r Platforms/SM8450/OnePlus10Pro/ Platforms/SM8450/MotoEdgePlus2022
```

### 2.3 Configure for hiphi

Edit the two main config files:

```bash
nano Platforms/SM8450/MotoEdgePlus2022/MotoEdgePlus2022.dsc
```

Key values to set:

```ini
# Resolution
gQcomTokenSpaceGuid.PcdMipiFrameBufferWidth|1080
gQcomTokenSpaceGuid.PcdMipiFrameBufferHeight|2400

# RAM
gQcomTokenSpaceGuid.PcdRamSize|0x200000000   # 8GB

# Panel — hiphi uses Samsung AMOLED (s6e3hc3 or similar)
# Pull exact DSI init sequence from Motorola kernel DTS:
# arch/arm64/boot/dts/vendor/qcom/dsi-panel-*hiphi*
```

To find the right panel init sequence, clone the Motorola kernel:

```bash
# In a separate dir — this is a large download
git clone --depth=1 https://github.com/MotorolaMobilityLLC/kernel-msm hiphi-kernel
grep -r "hiphi" hiphi-kernel/arch/arm64/boot/dts/ --include="*.dtsi" -l
```

### 2.4 Install Clang/LLVM Cross Compiler

```bash
sudo apt install -y clang lld llvm
clang --version   # Verify 14+ recommended
```

### 2.5 Build UEFI

```bash
cd ~/edk2-msm
./build.sh --device MotoEdgePlus2022 --release
```

Output: `Build/MotoEdgePlus2022/RELEASE_CLANGPDB/FV/uefi.img`

If build fails on `python3-distutils`:

```bash
sudo apt install -y python3-setuptools
```

### 2.6 Test Boot (Don’t Flash Yet)

```bash
# Phone must be in fastboot mode + USB attached via usbipd
fastboot boot Build/MotoEdgePlus2022/RELEASE_CLANGPDB/FV/uefi.img
```

First try will likely be a black screen. Iterate on display config until you see the Renegade logo or UEFI shell. Re-attach USB after each reboot:

```powershell
# PowerShell after each test reboot
usbipd attach --wsl --busid 2-3
```

### 2.7 Flash UEFI to Slot B (Safe)

Once the display initializes correctly:

```bash
# Keep Android on slot A, UEFI on slot B
fastboot --slot b flash boot Build/MotoEdgePlus2022/RELEASE_CLANGPDB/FV/uefi.img
```

Switch slots:

```bash
fastboot --set-active=b   # Boot Windows/UEFI
fastboot --set-active=a   # Boot Android
```

-----

## Stage 3 — Partition Layout

Done via ADB shell from WSL — TWRP must be running on the phone.

### 3.1 Find the UFS Block Device

```bash
adb shell lsblk | grep -E "sda|userdata"
# Typically /dev/block/sda or /dev/block/sda31 for userdata
```

### 3.2 Wipe and Resize userdata

```bash
adb shell

# Wipe userdata (this erases all Android user data)
mke2fs -t ext4 /dev/block/by-name/userdata

# Resize to leave space for Windows
parted /dev/block/sda
# Inside parted:
#   print                    <- find userdata partition number
#   resizepart N 400GB       <- shrink it (adjust to your needs)
#   quit
```

### 3.3 Create ESP and Windows Partitions

```bash
adb shell

DISK=/dev/block/sda

# ESP (FAT32, EFI System)
sgdisk -n 0:0:+512M -t 0:ef00 -c 0:"esp" $DISK

# Windows partition (NTFS formatted later from WinPE)
sgdisk -n 0:0:+55G -t 0:0700 -c 0:"win" $DISK

# Format ESP
mkfs.fat -F32 -s1 /dev/block/by-name/esp
```

Exit adb shell:

```bash
exit
```

-----

## Stage 4 — Deploy Windows ARM64

**Switch to Windows (PowerShell/CMD) for this stage.** WSL can’t run DISM or create WinPE bootable media.

### 4.1 Build ARM64 Windows Image (on Windows)

1. Go to [uupdump.net](https://uupdump.net)
1. Pick latest Windows 11 24H2, arch: `arm64`
1. Download the `.zip` conversion pack
1. Extract and run `uup_download_windows.cmd` — it downloads and builds `install.wim`

### 4.2 Create WinPE ARM64 (on Windows)

Install Windows ADK + WinPE add-on, then in CMD (admin):

```cmd
copype arm64 C:\WinPE_arm64
```

Add DISM and disk tools to WinPE:

```cmd
dism /Mount-Image /ImageFile:C:\WinPE_arm64\media\sources\boot.wim /Index:1 /MountDir:C:\WinPE_mount

dism /Image:C:\WinPE_mount /Add-Package /PackagePath:"C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\arm64\WinPE_OCs\WinPE-WMI.cab"

dism /Unmount-Image /MountDir:C:\WinPE_mount /Commit
```

Copy your `install.wim` to the WinPE media:

```cmd
mkdir C:\WinPE_arm64\media\sources_win
copy path\to\install.wim C:\WinPE_arm64\media\sources_win\
```

Write WinPE to a USB drive (replace `X:` with your drive letter):

```cmd
MakeWinPEMedia /UFD C:\WinPE_arm64 X:
```

### 4.3 Boot WinPE on the Phone

1. Switch phone to slot B (UEFI): `fastboot --set-active=b && fastboot reboot`
1. In the UEFI/Renegade menu, select USB boot
1. WinPE should load

> If UEFI doesn’t support USB boot yet, you can embed WinPE as a ramdisk in the UEFI image — more advanced, skip for now.

### 4.4 Deploy Windows from WinPE (on phone)

Inside WinPE CMD on the phone:

```cmd
diskpart
list disk
select disk 0
list part
:: Note partition numbers for 'win' and 'esp'
select part X
format fs=ntfs quick label="Windows"
assign letter=W
select part Y
assign letter=S
exit

dism /Apply-Image /ImageFile:D:\sources_win\install.wim /Index:1 /ApplyDir:W:\
bcdboot W:\Windows /s S: /f UEFI /l en-us
```

### 4.5 Configure BCD (on phone in WinPE)

```cmd
bcdedit /store S:\EFI\Microsoft\Boot\BCD /set {default} testsigning on
bcdedit /store S:\EFI\Microsoft\Boot\BCD /set {default} nointegritychecks on
bcdedit /store S:\EFI\Microsoft\Boot\BCD /set {default} recoveryenabled no
```

-----

## Stage 5 — Driver Porting

**This stage is split — driver extraction in WSL/Windows, offline injection from WinPE.**

### 5.1 Driver Status (SM8450 / hiphi)

|Component              |Status   |Notes                                     |
|-----------------------|---------|------------------------------------------|
|Display (framebuffer)  |✅ Works  |UEFI linear framebuffer                   |
|Display (full/HW accel)|⚠️ WIP    |Needs Adreno 730 WoA driver               |
|Touch                  |❌ None   |Focaltech — no WoA driver exists          |
|USB-C                  |✅ Works  |Functional in most SM8450 UEFI ports      |
|Wi-Fi (WCN6750)        |⚠️ Partial|Inconsistent; USB dongle more reliable    |
|Bluetooth              |❌ None   |                                          |
|Audio                  |❌ None   |                                          |
|Cellular               |❌ None   |Not expected in WoA                       |
|Camera                 |❌ None   |                                          |
|Battery / ACPI         |⚠️ Partial|Basic power states only                   |
|GPU (Adreno 730)       |⚠️ WIP    |No public WoA driver; software render only|
|Sensors / IMU          |❌ None   |                                          |

### 5.2 Extract Reference Drivers in WSL

The Lenovo ThinkPad X13s Gen 1 uses SM8450 (SC8280XP) and ships with official Windows drivers that may partially apply.

```bash
# Download the X13s driver pack from Lenovo's site, then extract in WSL
mkdir -p ~/hiphi-drivers
cd ~/hiphi-drivers

# Mount a Windows driver .exe or .cab with 7z
7z x lenovo-x13s-drivers.exe -o./extracted/
```

Or extract from a running Windows install on the ThinkPad:

```powershell
# On Windows host
pnputil /export-driver * C:\extracted-drivers\
```

Then copy to WSL:

```bash
cp -r /mnt/c/extracted-drivers/ ~/hiphi-drivers/
```

### 5.3 Inject Drivers Offline from WinPE

Back in WinPE on the phone, with drivers on USB:

```cmd
dism /Image:W:\ /Add-Driver /Driver:D:\drivers\ /Recurse /ForceUnsigned
```

Or post-first-boot from an elevated CMD:

```cmd
pnputil /add-driver C:\drivers\*.inf /install /subdirs
```

### 5.4 Check WOA-Project for SM8450 Packs

```bash
# In WSL
git clone https://github.com/WOA-Project/Qualcomm-Reference-Drivers.git
ls Qualcomm-Reference-Drivers/
# Look for SM8450 or WCN6750 entries
```

-----

## Stage 6 — First Boot & OOBE

### 6.1 Reboot to Windows

```bash
# WSL — switch to slot B and reboot
fastboot --set-active=b
fastboot reboot
```

UEFI → Windows Boot Manager → spinning dots. If it BSODs, common early causes:

|BSOD Code                            |Likely Cause                                    |
|-------------------------------------|------------------------------------------------|
|`INACCESSIBLE_BOOT_DEVICE`           |Wrong partition letter in BCD or bad NTFS format|
|`SYSTEM_THREAD_EXCEPTION_NOT_HANDLED`|Bad driver injected                             |
|`PAGE_FAULT_IN_NONPAGED_AREA`        |RAM config wrong in UEFI DSC                    |
|Reboot loop                          |UEFI not passing memory map correctly           |

### 6.2 OOBE Tips

- **No touch** — Use a USB-C hub + USB mouse. Mandatory until touch drivers exist.
- **No Wi-Fi** — Use USB-C Ethernet adapter or a USB Wi-Fi dongle with ARM64 driver support.
- Choose **“I don’t have internet”** during OOBE for a local account.
- Skip automatic updates for now.

### 6.3 Post-Install from WSL ADB

If the phone is on Windows and ADB is running (you can enable it in Windows Settings → Developer):

```bash
# Re-attach in usbipd, then
adb shell
# Run powershell commands remotely
adb shell powershell "bcdedit /set testsigning on"
```

-----

## Switching Between Android and Windows

```bash
# Boot Android
fastboot --set-active=a && fastboot reboot

# Boot Windows/UEFI
fastboot --set-active=b && fastboot reboot
```

Keep this as the workflow during development so you’re never locked out.

-----

## usbipd Quick Reference

You’ll run these constantly. Keep a PowerShell window open as admin:

```powershell
# List all USB devices
usbipd list

# Bind once per device (first time only)
usbipd bind --busid 2-3

# Attach to WSL (run after every replug or mode switch)
usbipd attach --wsl --busid 2-3

# Detach (hand back to Windows)
usbipd detach --busid 2-3
```

> The phone changes BUSID when switching between ADB mode and fastboot mode on some USB controllers. Run `usbipd list` again if attach fails.

-----

## Known Limitations

- **No touch** — Focaltech/Goodix SM8450 panel has no WoA driver. USB mouse required.
- **No GPU accel** — Adreno 730 WoA drivers are not publicly available. Software rendering only.
- **No cellular** — Expected; Windows doesn’t use Snapdragon modem stack.
- **Wi-Fi unreliable** — WCN6750 WoA support is partial. USB dongle recommended.
- **Battery drain** — ACPI power management incomplete; expect poor battery life.
- **Secure Boot disabled** — Required. Motorola’s chain won’t trust Renegade UEFI.
- **5G UW** — Non-functional. mmWave is Android-only.
- **usbipd friction** — You’ll need to re-attach after every USB mode switch.

-----

## Resources

|Resource               |URL                                                                         |
|-----------------------|----------------------------------------------------------------------------|
|Renegade Project EDK2  |<https://github.com/edk2-porting/edk2-msm>                                  |
|WOA-Project            |<https://github.com/WOA-Project>                                            |
|usbipd-win             |<https://github.com/dorssel/usbipd-win>                                     |
|UUP Dump (ARM64 ISO)   |<https://uupdump.net>                                                       |
|Windows ADK            |<https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install>|
|SM8450 Linux kernel    |<https://github.com/msm-mainline/linux>                                     |
|postmarketOS hiphi     |<https://wiki.postmarketos.org/wiki/Motorola_Edge+_(2022)>                  |
|Motorola RETUS firmware|<https://mirrors.lolinet.com/firmware/motorola/>                            |
|Windows on ARM Discord |Search “windows on qualcomm”                                                |

-----

*Guide for Moto Edge+ 2022 UW (XT2201-4) — SM8450 / hiphi — WSL Edition — May 2026*
*Status: Active bring-up / pioneering port*
