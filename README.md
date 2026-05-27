# Windows 11 on Moto Edge+ 2022 UW (XT2201-4)
## Complete Porting Guide — WSL2 Edition

> ⚠️ **PIONEER TERRITORY** — No complete WoA port exists for `hiphi` as of mid-2026.
> This guide walks through every command from a fresh Windows PC to a booting Windows 11 install on the phone.
> **You can hard brick this device.** Read every warning. Do every backup. Proceed only if you accept that risk.

---

## Table of Contents

1. [Device Reference](#1-device-reference)
2. [How This Works — Big Picture](#2-how-this-works--big-picture)
3. [WSL2 Setup](#3-wsl2-setup)
4. [USB Passthrough — usbipd](#4-usb-passthrough--usbipd)
5. [Stage 1 — Enable Developer Options & ADB](#5-stage-1--enable-developer-options--adb)
6. [Stage 2 — Bootloader Unlock](#6-stage-2--bootloader-unlock)
7. [Stage 3 — Back Up Critical Partitions](#7-stage-3--back-up-critical-partitions)
8. [Stage 4 — Flash TWRP](#8-stage-4--flash-twrp)
9. [Stage 5 — Build the UEFI Firmware](#9-stage-5--build-the-uefi-firmware)
10. [Stage 6 — Partition the Storage](#10-stage-6--partition-the-storage)
11. [Stage 7 — Build Windows ARM64 Image](#11-stage-7--build-windows-arm64-image)
12. [Stage 8 — Build WinPE ARM64](#12-stage-8--build-winpe-arm64)
13. [Stage 9 — Deploy Windows from WinPE](#13-stage-9--deploy-windows-from-winpe)
14. [Stage 10 — Collect and Inject Drivers](#14-stage-10--collect-and-inject-drivers)
15. [Stage 11 — First Boot and OOBE](#15-stage-11--first-boot-and-oobe)
16. [Dual Boot Reference](#16-dual-boot-reference)
17. [usbipd Quick Reference](#17-usbipd-quick-reference)
18. [Troubleshooting](#18-troubleshooting)
19. [Driver Status Table](#19-driver-status-table)
20. [Resources](#20-resources)

---

## 1. Device Reference

| Field | Value |
|---|---|
| Device name | Motorola Edge+ 2022 UW |
| Codename | `hiphi` |
| Target variant | **XT2201-4 (Verizon UW)** |
| Other variants | XT2201-1 (global), XT2201-3 (Boost Mobile) |
| SoC | Qualcomm Snapdragon 8 Gen 1 (SM8450) |
| CPU | 1x Cortex-X2 @ 3.0GHz, 3x A710 @ 2.5GHz, 4x A510 @ 1.8GHz |
| GPU | Adreno 730 |
| RAM | 8GB LPDDR5 |
| Storage | 512GB UFS 3.1 |
| Display | 6.7" Samsung pOLED, 1080×2400, 144Hz |
| Wi-Fi | Qualcomm WCN6750 (Wi-Fi 6E) |
| Bluetooth | 5.2 |
| USB | USB 3.1 Gen 2 (USB-C) |
| Boot slots | A/B (we use A for Android, B for UEFI/Windows) |
| Bootloader | Motorola fastboot, Secure Boot enforced |
| Touch panel | Focaltech FT8726 |
| Audio codec | WCD9380 |

---

## 2. How This Works — Big Picture

Normal Android uses the Android bootloader → Android kernel. To boot Windows we need to replace the boot slot with a UEFI firmware (Renegade/EDK2), which is what PC motherboards run. UEFI then boots Windows Boot Manager, which loads Windows ARM64.

```
[Phone powers on]
       ↓
[Motorola ABL (always present)]
       ↓
[Slot B] → [Renegade UEFI]
       ↓
[Windows Boot Manager (EFI partition)]
       ↓
[Windows 11 ARM64 (win partition)]
```

We keep Android untouched on Slot A. Switching between them is one fastboot command.

### What runs on WSL vs Windows

| Task | Where |
|---|---|
| Build EDK2 UEFI | WSL (Linux toolchain) |
| ADB and fastboot commands | WSL (via usbipd) |
| Partition the phone | WSL (adb shell → sgdisk on device) |
| Extract kernel DTS | WSL |
| Collect drivers | WSL + Windows |
| Build ARM64 Windows image | Windows (UUP Dump script) |
| Create WinPE boot media | Windows (ADK tools) |
| Deploy Windows with DISM | WinPE on the phone |
| BCD configuration | WinPE on the phone |

---

## 3. WSL2 Setup

### 3.1 Install WSL2

Open **PowerShell as Administrator** (right-click Start → Terminal (Admin)):

```powershell
# Install WSL2 with Ubuntu 24.04
wsl --install -d Ubuntu-24.04

# Make sure WSL2 is the default version
wsl --set-default-version 2
```

**Reboot when prompted.** After reboot, a terminal window opens automatically to finish Ubuntu setup. Create a UNIX username and password — this is your WSL user, not your Windows login.

Verify WSL2 is running:
```powershell
wsl -l -v
# Expected output:
#   NAME            STATE           VERSION
# * Ubuntu-24.04    Running         2
```

If VERSION shows 1, upgrade it:
```powershell
wsl --set-version Ubuntu-24.04 2
```

### 3.2 Open the WSL Terminal

Search "Ubuntu" in the Start menu and open it. All `bash` commands in this guide run here unless labeled `[Windows CMD]` or `[PowerShell]`.

### 3.3 Update the System

```bash
sudo apt update && sudo apt upgrade -y
```

### 3.4 Install Every Tool You'll Need

```bash
sudo apt install -y \
  git \
  curl \
  wget \
  nano \
  build-essential \
  uuid-dev \
  iasl \
  nasm \
  clang \
  clang-14 \
  lld \
  lld-14 \
  llvm \
  python3 \
  python3-setuptools \
  python3-distutils-extra \
  android-tools-adb \
  android-tools-fastboot \
  sgdisk \
  gdisk \
  parted \
  dosfstools \
  e2fsprogs \
  p7zip-full \
  wimtools \
  gcc-aarch64-linux-gnu \
  binutils-aarch64-linux-gnu \
  device-tree-compiler \
  libssl-dev \
  flex \
  bison \
  bc \
  rename
```

Pin Clang 14 as the default compiler (Renegade builds best with it):
```bash
sudo update-alternatives --install /usr/bin/clang clang /usr/bin/clang-14 100
sudo update-alternatives --install /usr/bin/lld lld /usr/bin/lld-14 100
sudo update-alternatives --install /usr/bin/ld.lld ld.lld /usr/bin/ld.lld-14 100
```

Verify everything installed correctly:
```bash
echo "--- Tool Versions ---"
adb version
fastboot --version
clang --version | head -1
nasm --version | head -1
python3 --version
git --version
sgdisk --version | head -1
echo "--- All good ---"
```

### 3.5 Create Your Working Directory

```bash
mkdir -p ~/woa-hiphi/{backups,drivers,firmware,logs}
cd ~/woa-hiphi
echo "Working directory: $(pwd)"
```

---

## 4. USB Passthrough — usbipd

WSL2 is a virtual machine and cannot access USB devices by default. `usbipd-win` forwards USB devices from Windows into the WSL2 VM.

### 4.1 Install usbipd-win on Windows

Open **PowerShell as Administrator**:

```powershell
winget install --interactive --exact dorssel.usbipd-win
```

> Note: the package ID is `dorssel.usbipd-win` (not `dorsberg`). **Reboot after install.**

### 4.2 Install the usbip Client in WSL

```bash
sudo apt install -y linux-tools-generic hwdata

# Find the correct usbip binary path and register it
USBIP_PATH=$(ls /usr/lib/linux-tools/*/usbip 2>/dev/null | head -1)
echo "Found usbip at: $USBIP_PATH"

sudo update-alternatives --install /usr/local/bin/usbip usbip "$USBIP_PATH" 20

# Verify
usbip version
```

### 4.3 Bind the Phone (One-Time Per Device)

Plug the phone into your PC. Make sure it's in **normal Android mode** with USB debugging enabled for now.

In **PowerShell (admin)**:

```powershell
# List all USB devices
usbipd list

# Example output:
# BUSID  VID:PID    DEVICE                             STATE
# 1-1    0781:5590  SanDisk USB                        Not shared
# 2-3    22b8:2e81  Motorola PCS Motorola Phone (ADB)  Not shared
# 3-1    8087:0033  Intel Wireless Bluetooth           Not shared
```

Find your phone (Motorola). Note its `BUSID` — in this example `2-3`.

```powershell
# Bind the phone (one-time setup per device)
usbipd bind --busid 2-3
```

After binding, state changes from `Not shared` to `Shared`.

> **If you reboot Windows**, you may need to re-bind. Just run `usbipd bind --busid 2-3` again.

### 4.4 Attach to WSL

```powershell
usbipd attach --wsl --busid 2-3
```

Switch to WSL and verify:
```bash
# Check USB devices visible in WSL
lsusb
# Should show something like: Bus 001 Device 002: ID 22b8:2e81 Motorola PCS

# Check ADB sees the phone
adb devices
# Expected:
# List of devices attached
# ZY224MZ6BW    device
```

If the phone shows `unauthorized` instead of `device`, accept the RSA fingerprint prompt on the phone screen.

### 4.5 The Most Important Rule

**Every time the phone reboots or you switch between ADB mode and fastboot mode, the device detaches from WSL.** You must re-run the attach command:

```powershell
# Run this in PowerShell (admin) after EVERY phone reboot or mode switch
usbipd attach --wsl --busid 2-3
```

The BUSID stays the same as long as you use the same USB port. If you switch ports, run `usbipd list` to find the new BUSID.

---

## 5. Stage 1 — Enable Developer Options & ADB

### 5.1 Enable Developer Options

On the phone, go through Settings:
```
Settings
  → About Phone
    → Software Information
      → Build Number    (tap this 7 times)
```

You'll see "You are now a developer!"

### 5.2 Enable USB Debugging and OEM Unlocking

```
Settings
  → System
    → Developer Options
      → USB Debugging    (toggle ON)
      → OEM Unlocking    (toggle ON if not greyed out)
```

> On the Verizon UW (XT2201-4), **OEM Unlocking will be greyed out** until you flash RETUS firmware. We handle that in Stage 2.

### 5.3 Verify ADB Works

With the phone attached (usbipd), in WSL:

```bash
# Check device is listed
adb devices

# If it shows "unauthorized", unlock it on the phone screen first
# Then check again
adb devices

# Should show:
# List of devices attached
# XXXXXXXXXX    device

# Test a basic command
adb shell getprop ro.product.device
# Expected output: hiphi

adb shell getprop ro.product.model
# Expected output: motorola edge+ 5G UW (2022)
```

---

## 6. Stage 2 — Bootloader Unlock

The XT2201-4 Verizon variant ships with the bootloader locked and OEM Unlocking grayed out. You need to flash RETUS (Retail US Unlocked) firmware to lift this restriction.

### 6.1 Download RETUS Firmware

In WSL:
```bash
cd ~/woa-hiphi/firmware

# Open the Motorola mirror directory listing
curl -s "https://mirrors.lolinet.com/firmware/motorola/hiphi/official/RETUS/" \
  | grep -oP 'href="[^"]+\.zip"' | sed 's/href="//;s/"//'
```

This prints available firmware zip filenames. Pick the latest one. Download it:

```bash
# Replace FILENAME with the actual filename from above
RETUS_FILE="XT2201-4_RETUS_FILENAME_HERE.zip"

wget "https://mirrors.lolinet.com/firmware/motorola/hiphi/official/RETUS/$RETUS_FILE" \
  -O ~/woa-hiphi/firmware/retus.zip

# Verify the download completed
ls -lh ~/woa-hiphi/firmware/retus.zip
```

### 6.2 Extract the Firmware

```bash
cd ~/woa-hiphi/firmware
unzip retus.zip -d retus/
ls retus/
```

You should see files like: `bootloader.img`, `radio.img`, `boot.img`, `vbmeta.img`, etc.

### 6.3 Flash RETUS Firmware

> This does **NOT** wipe user data. It only replaces system firmware.

Reboot the phone to fastboot:
```bash
adb reboot bootloader
```

**Re-attach USB immediately** (PowerShell admin):
```powershell
usbipd attach --wsl --busid 2-3
```

Wait for the phone to show the fastboot screen (white text on black), then verify in WSL:
```bash
fastboot devices
# Should show your device serial number
```

Flash the RETUS firmware files:
```bash
cd ~/woa-hiphi/firmware/retus/

# Flash bootloader
fastboot flash bootloader bootloader.img

# Reboot bootloader to apply
fastboot reboot bootloader
```

Re-attach USB:
```powershell
usbipd attach --wsl --busid 2-3
```

```bash
# Flash radio/modem
fastboot flash radio radio.img

# Flash other partitions if present
# (check what's in your retus/ folder and flash accordingly)
[ -f vbmeta.img ] && fastboot flash vbmeta --disable-verity --disable-verification vbmeta.img
[ -f vbmeta_system.img ] && fastboot flash vbmeta_system vbmeta_system.img

# Reboot to Android
fastboot reboot
```

Re-attach USB after reboot, complete any Android first-time setup prompts, then go back to:
```
Settings → System → Developer Options → OEM Unlocking
```
It should now be toggleable. Enable it.

### 6.4 Unlock the Bootloader

> **This wipes all data on the phone.** Back up anything important before this step.

```bash
adb reboot bootloader
```

Re-attach:
```powershell
usbipd attach --wsl --busid 2-3
```

```bash
# Verify unlock is available
fastboot oem device-info
# Look for: "(bootloader) Device unlocking enabled: true"

# Unlock
fastboot oem unlock
```

A warning screen appears on the phone. Use the **volume keys** to move to "Unlock the bootloader" and press **Power** to confirm.

The phone will wipe and reboot. Let it finish the wipe and reach the Android setup screen.

```bash
# After phone reboots — re-attach USB
# (PowerShell): usbipd attach --wsl --busid 2-3

# Verify unlock was successful
adb reboot bootloader
# (PowerShell): usbipd attach --wsl --busid 2-3

fastboot oem device-info
# Should now show: "(bootloader) Device unlocked: true"

fastboot reboot
```

Complete the minimal Android setup (skip Google account, skip everything). Re-enable Developer Options and USB Debugging.

---

## 7. Stage 3 — Back Up Critical Partitions

**Do not skip this.** These partitions store per-device calibration data for the cellular modem and Wi-Fi antenna. If lost, your phone loses cellular and Wi-Fi permanently.

### 7.1 Create Backup Directory

```bash
mkdir -p ~/woa-hiphi/backups
```

### 7.2 Pull All Critical Partitions

```bash
# Helper function to back up a partition
backup_partition() {
  local NAME=$1
  echo "Backing up $NAME..."
  adb shell "dd if=/dev/block/by-name/$NAME of=/sdcard/${NAME}.img bs=4096 2>/dev/null"
  adb pull "/sdcard/${NAME}.img" ~/woa-hiphi/backups/
  adb shell "rm /sdcard/${NAME}.img"
  echo "  → Done: $(ls -lh ~/woa-hiphi/backups/${NAME}.img 2>/dev/null | awk '{print $5}')"
}

# Back up each partition
backup_partition persist
backup_partition modemst1
backup_partition modemst2
backup_partition fsg
backup_partition nvm
backup_partition abl_a
backup_partition abl_b
backup_partition xbl_a
backup_partition xbl_b
backup_partition tz_a
backup_partition tz_b
backup_partition hyp_a
backup_partition hyp_b
```

### 7.3 Verify All Backups Have Size

```bash
echo "=== Backup Sizes ==="
ls -lh ~/woa-hiphi/backups/
echo "Total:"
du -sh ~/woa-hiphi/backups/
```

Every file should be larger than 0 bytes. If any are 0 bytes, re-run that backup step.

### 7.4 Copy Backups to Windows

```bash
# Copy to your Windows Documents folder
WINUSER=$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d '\r')
mkdir -p "/mnt/c/Users/$WINUSER/Documents/hiphi-backups"
cp -r ~/woa-hiphi/backups/ "/mnt/c/Users/$WINUSER/Documents/hiphi-backups/"
echo "Copied to Windows: /mnt/c/Users/$WINUSER/Documents/hiphi-backups/"
```

Also copy them to a USB drive or cloud storage. Treat these like BIOS firmware for the phone.

---

## 8. Stage 4 — Flash TWRP

TWRP (Team Win Recovery Project) gives you a Linux shell with full block device access. You need it to manage partitions in Stage 6.

### 8.1 Get TWRP for hiphi

```bash
cd ~/woa-hiphi

# Check for XT2201-4 specific build first
# If none exists, the XT2201-1 (global) build works — same hardware

# Try downloading from TWRP's official server
wget -q --spider https://dl.twrp.me/hiphi/ && \
  wget https://dl.twrp.me/hiphi/twrp-hiphi.img -O twrp-hiphi.img || \
  echo "Not on TWRP server — check XDA or GitHub for a hiphi TWRP build"
```

If the download fails, check XDA Developers:
- Search: `site:forum.xda-developers.com TWRP hiphi XT2201`
- Download the `.img` file and copy it to WSL:

```bash
# Copy from Windows Downloads to WSL
WINUSER=$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d '\r')
cp "/mnt/c/Users/$WINUSER/Downloads/twrp-hiphi.img" ~/woa-hiphi/twrp-hiphi.img
```

### 8.2 Flash TWRP

```bash
cd ~/woa-hiphi

adb reboot bootloader
```

Re-attach USB:
```powershell
usbipd attach --wsl --busid 2-3
```

```bash
# Verify fastboot sees the device
fastboot devices

# Flash TWRP to the recovery partition
fastboot flash recovery twrp-hiphi.img

# Reboot directly into recovery (hold Vol Up + Power when screen turns on)
fastboot reboot recovery
```

Re-attach after reboot:
```powershell
usbipd attach --wsl --busid 2-3
```

### 8.3 Verify TWRP Is Running

```bash
# Check device is in recovery mode
adb devices
# Should show: XXXXXXXXXX    recovery

# Test shell access
adb shell ls /sdcard/
adb shell cat /proc/version
# Should show a Linux kernel version

# Confirm block devices are accessible
adb shell ls /dev/block/by-name/ | head -20
```

---

## 9. Stage 5 — Build the UEFI Firmware

This is the hardest and most iterative stage. You're compiling a full UEFI firmware image from source that can boot the SM8450 hardware and hand control to Windows Boot Manager.

### 9.1 Clone Renegade EDK2

```bash
cd ~/woa-hiphi
git clone https://github.com/edk2-porting/edk2-msm.git
cd edk2-msm

# Pull all submodules (this takes several minutes)
git submodule update --init --recursive

echo "Clone complete. Submodule count: $(git submodule | wc -l)"
```

### 9.2 Check Existing SM8450 Device Configs

```bash
ls Platforms/SM8450/
```

You'll see something like:
```
OnePlus10Pro/  Samsung_Galaxy_S22/  Xiaomi12/
```

`hiphi` is not there. You'll create it by copying the closest device. OnePlus 10 Pro is also SM8450 and a good reference.

### 9.3 Clone the Motorola hiphi Kernel

You need the device tree source (DTS) files from Motorola's kernel to get the correct display panel initialization sequence, memory map, and GPIO assignments.

```bash
cd ~/woa-hiphi

# Clone with shallow depth to save time/space
git clone --depth=1 \
  --branch android-msm-hiphi-5.10-android12-qpr3 \
  https://github.com/MotorolaMobilityLLC/kernel-msm \
  hiphi-kernel

# If that branch doesn't exist, find the right one:
# git ls-remote https://github.com/MotorolaMobilityLLC/kernel-msm | grep hiphi
```

Find the hiphi-specific DTS files:
```bash
echo "=== hiphi DTS files ==="
find hiphi-kernel/arch/arm64/boot/dts -name "*.dts" -o -name "*.dtsi" | \
  xargs grep -l "hiphi\|XT2201\|edge.*plus\|edge.*2022" 2>/dev/null

echo ""
echo "=== Display panel DTS files ==="
find hiphi-kernel/arch/arm64/boot/dts -name "dsi-panel*" | head -20
```

Find the exact panel for hiphi:
```bash
# Look for hiphi-specific panel references
grep -r "dsi_panel\|dsi-panel" \
  hiphi-kernel/arch/arm64/boot/dts \
  --include="*.dtsi" --include="*.dts" | \
  grep -i "hiphi" | head -10

# Find panel resolution info
grep -r "1080\|2400\|s6e3\|samsung" \
  hiphi-kernel/arch/arm64/boot/dts \
  --include="dsi-panel*.dtsi" | head -20
```

Save the panel filename — you'll need it in the UEFI config.

### 9.4 Create the hiphi Device Config

```bash
cd ~/woa-hiphi/edk2-msm

# Copy OnePlus 10 Pro config as a base
cp -r Platforms/SM8450/OnePlus10Pro/ Platforms/SM8450/MotoEdgePlus2022

cd Platforms/SM8450/MotoEdgePlus2022/

# Rename all files from OnePlus10Pro to MotoEdgePlus2022
for f in *OnePlus10Pro*; do
  mv "$f" "${f//OnePlus10Pro/MotoEdgePlus2022}"
done

# Also rename references inside the files
find . -type f -exec sed -i 's/OnePlus10Pro/MotoEdgePlus2022/g' {} \;
find . -type f -exec sed -i 's/OnePlus 10 Pro/Motorola Edge+ 2022 UW/g' {} \;

echo "Files in config directory:"
ls -la
```

### 9.5 Edit MotoEdgePlus2022.dsc

```bash
nano MotoEdgePlus2022.dsc
```

Find and update these values (search with Ctrl+W in nano):

```ini
[Defines]
  PLATFORM_NAME                  = MotoEdgePlus2022
  PLATFORM_GUID                  = REPLACE_WITH_NEW_GUID_SEE_BELOW
  PLATFORM_VERSION               = 0.1

[PcdsFixedAtBuild.common]
  # --- Display ---
  # hiphi is 1080x2400 pOLED
  gQcomTokenSpaceGuid.PcdMipiFrameBufferWidth|1080
  gQcomTokenSpaceGuid.PcdMipiFrameBufferHeight|2400
  gQcomTokenSpaceGuid.PcdMipiFrameBufferPixelBpp|32

  # --- RAM ---
  # 8GB = 0x200000000
  gQcomTokenSpaceGuid.PcdRamSize|0x200000000

  # --- Storage ---
  # UFS = 1
  gQcomTokenSpaceGuid.PcdStorageType|1
```

Generate a new unique GUID:
```bash
python3 -c "import uuid; print(uuid.uuid4())"
```

Copy that output and paste it into the PLATFORM_GUID line.

Save (Ctrl+X → Y → Enter).

### 9.6 Bulk Replace References in FDF File

```bash
# Replace all platform name references in the FDF
sed -i 's/OnePlus10Pro/MotoEdgePlus2022/g' MotoEdgePlus2022.fdf

# Verify
grep "MotoEdgePlus2022" MotoEdgePlus2022.fdf | head -5
```

### 9.7 Register the New Device in the Build System

```bash
cd ~/woa-hiphi/edk2-msm

# Check if there's a device registry file that needs updating
cat Platforms/SM8450/SM8450.conf 2>/dev/null || \
cat Platforms/SM8450/PlatformList.txt 2>/dev/null || \
  echo "No central registry found — device config is self-contained"

# Some builds use a platforms list:
if [ -f Platforms/SM8450/PlatformList.txt ]; then
  echo "MotoEdgePlus2022" >> Platforms/SM8450/PlatformList.txt
fi
```

### 9.8 Build the UEFI Image

```bash
cd ~/woa-hiphi/edk2-msm

# Run the build
./build.sh --device MotoEdgePlus2022 --release 2>&1 | tee ~/woa-hiphi/logs/uefi-build-1.log
```

**If the build fails**, check the error:

```bash
# View the last 50 lines of the build log
tail -50 ~/woa-hiphi/logs/uefi-build-1.log
```

Common fixes:

```bash
# Fix: python3 distutils missing
sudo apt install -y python3-setuptools python3-distutils-extra

# Fix: clang version mismatch
sudo apt install -y clang-14 lld-14
sudo update-alternatives --set clang /usr/bin/clang-14
sudo update-alternatives --set lld /usr/bin/lld-14

# Fix: missing iasl
sudo apt install -y iasl

# Fix: missing nasm
sudo apt install -y nasm

# Fix: uuid-dev missing
sudo apt install -y uuid-dev
```

After fixing, re-run the build:
```bash
./build.sh --device MotoEdgePlus2022 --release 2>&1 | tee ~/woa-hiphi/logs/uefi-build-2.log
```

A successful build ends with something like:
```
- Done -
Build end time: ...
- Done -
```

And produces:
```bash
ls -lh Build/MotoEdgePlus2022/RELEASE_CLANGPDB/FV/uefi.img
# Should show a file around 2-6MB
```

### 9.9 Test Boot the UEFI Without Flashing

`fastboot boot` loads an image into RAM for one boot only. Nothing is written to storage. This is how you iterate safely.

```bash
adb reboot bootloader
```

Re-attach:
```powershell
usbipd attach --wsl --busid 2-3
```

```bash
fastboot devices
# Make sure device is detected

# Test boot — this does NOT flash anything
fastboot boot ~/woa-hiphi/edk2-msm/Build/MotoEdgePlus2022/RELEASE_CLANGPDB/FV/uefi.img
```

**What you might see:**

| Result | Meaning | Fix |
|---|---|---|
| Black screen, phone reboots after ~30s | Display init failed | Check DSI panel config |
| Garbled / wrong colors | Wrong pixel format | Try `PcdMipiFrameBufferPixelBpp|24` |
| Correct resolution but corrupted | Wrong framebuffer address | Check memory base in kernel DTS |
| Renegade logo visible | ✅ Display works | Proceed to flash |
| UEFI shell prompt | ✅ Full success | Proceed to flash |

**If you get a black screen**, find the correct display base address from the kernel DTS:

```bash
grep -r "mdss_mdp\|mdp_base\|fb_base\|framebuffer_base" \
  ~/woa-hiphi/hiphi-kernel/arch/arm64/boot/dts \
  --include="*.dtsi" | head -10

# Also check the memory map for display
grep -r "qcom,mdss_mdp" \
  ~/woa-hiphi/hiphi-kernel/arch/arm64/boot/dts \
  --include="*.dtsi" | head -5
```

Update the `.dsc` with whatever address you find, rebuild, and test boot again. Repeat until you see the Renegade logo.

After each test, the phone will reboot to Android (or fastboot). Re-attach USB each time:
```powershell
usbipd attach --wsl --busid 2-3
```

### 9.10 Flash UEFI to Slot B

Once the UEFI displays the logo correctly:

```bash
adb reboot bootloader
```

Re-attach:
```powershell
usbipd attach --wsl --busid 2-3
```

```bash
# Flash UEFI to slot B only — slot A stays Android
fastboot --slot b flash boot \
  ~/woa-hiphi/edk2-msm/Build/MotoEdgePlus2022/RELEASE_CLANGPDB/FV/uefi.img

# Verify the flash succeeded
echo "Flash status: $?"

# Check what's on slot B
fastboot --slot b getvar slot-suffixes 2>/dev/null || true
```

You can now switch to slot B to boot UEFI:
```bash
fastboot --set-active=b
fastboot reboot
# (Re-attach USB after reboot)
```

And switch back to Android:
```bash
fastboot --set-active=a
fastboot reboot
```

---

## 10. Stage 6 — Partition the Storage

You need to carve space out of `userdata` to create an EFI System Partition (ESP) and a Windows data partition.

> **This wipes Android user data again.** Make sure backups from Stage 3 are safe.

### 10.1 Boot to TWRP

```bash
adb reboot recovery
```

Re-attach:
```powershell
usbipd attach --wsl --busid 2-3
```

Verify TWRP shell works:
```bash
adb shell whoami
# Should output: root

adb shell uname -r
# Should show a kernel version
```

### 10.2 Identify the Block Device Layout

```bash
# See all block devices
adb shell lsblk

# Find userdata specifically
adb shell readlink /dev/block/by-name/userdata
# Outputs something like: /dev/block/sda31

# Find the parent disk (strip partition number)
adb shell ls -la /dev/block/by-name/ | grep -E "sda$|mmcblk0$"
```

The main UFS storage is usually `/dev/block/sda`. Let's confirm:

```bash
adb shell sgdisk -p /dev/block/sda 2>/dev/null | head -30
# Should show a GPT partition table with ~30+ Android partitions
```

Save the current partition table as a reference:
```bash
adb shell sgdisk -p /dev/block/sda > ~/woa-hiphi/logs/original-partitions.txt 2>&1
cat ~/woa-hiphi/logs/original-partitions.txt
```

**Note the userdata partition number** (usually the last one, e.g. partition 31) and its start sector.

### 10.3 Wipe userdata

```bash
# Format userdata as ext4 to free up space cleanly
adb shell mke2fs -t ext4 /dev/block/by-name/userdata
```

### 10.4 Shrink userdata to Make Room for Windows

```bash
adb shell
```

You're now in the TWRP shell on the phone. Run:

```bash
# Inside adb shell — run on the phone
parted /dev/block/sda

# Inside parted:
print
# Note the userdata partition number — let's call it N
# Note the current end position — e.g. 512GB

# Shrink userdata (leaves ~58GB for Windows + ESP)
# Example: if userdata starts at 50GB and disk ends at 512GB:
resizepart N 450GB
# (Adjust 450GB based on your actual disk and desired sizes)

# Confirm the change
print

quit
```

Exit the adb shell:
```bash
exit
```

### 10.5 Create the ESP and Windows Partitions

```bash
adb shell
```

Inside the phone shell:
```bash
DISK=/dev/block/sda

# Verify the disk variable is right
echo "Disk: $DISK"
ls $DISK

# Create EFI System Partition — 512MB, type ef00 = EFI System
sgdisk -n 0:0:+512M -t 0:ef00 -c 0:"esp" $DISK

# Create Windows partition — 55GB, type 0700 = Microsoft Basic Data
sgdisk -n 0:0:+55G -t 0:0700 -c 0:"win" $DISK

# Verify the new partitions were created
sgdisk -p $DISK | tail -10

# Check they show up in by-name
ls /dev/block/by-name/ | grep -E "esp|win"
```

### 10.6 Format the ESP

```bash
# Still inside adb shell on the phone

# Format as FAT32 (-F32), 1 sector per cluster (-s1)
mkfs.fat -F32 -s1 /dev/block/by-name/esp

# Verify it formatted correctly
blkid /dev/block/by-name/esp
# Should show: TYPE="vfat"

exit
```

> The Windows partition (`win`) gets formatted from WinPE as NTFS in Stage 9. Don't format it yet.

### 10.7 Verify Partitions from WSL

```bash
# Double-check the new partitions are visible
adb shell "ls -la /dev/block/by-name/ | grep -E 'esp|win'"

# Check ESP was formatted
adb shell "blkid /dev/block/by-name/esp"
```

---

## 11. Stage 7 — Build Windows ARM64 Image

**Switch to Windows for this stage.** WSL cannot run the UUP Dump scripts (they need PowerShell and Windows update infrastructure).

### 11.1 Download the UUP Dump Script Package

Open a browser on your Windows PC and go to **[uupdump.net](https://uupdump.net)**:

1. Click **"Latest Dev/Beta/Release Preview build"**
2. On the search results, find **Windows 11** — pick the newest build
3. Under **Architecture**, select `arm64`
4. Click the build name
5. Select language: `English (United States)`
6. Click **Next**
7. Under editions, check: `Windows 11 Home` and `Windows 11 Pro`
8. Click **Next**
9. Download method: **Download and convert using aria2**
10. Click **Create download package** → downloads a `.zip`

### 11.2 Run the Conversion Script

Extract the downloaded `.zip`. Inside you'll find `uup_download_windows.cmd`.

Open **CMD as Administrator** and run:

```cmd
cd C:\Users\%USERNAME%\Downloads\uup_download_windows
uup_download_windows.cmd
```

This will:
- Download all Windows ARM64 install files from Microsoft's servers
- Assemble them into an `install.wim` image

Takes 20–40 minutes. Needs ~10GB free disk space. The `install.wim` ends up in the same folder when done.

Verify it was built:
```cmd
dir install.wim
:: Should show a file around 4-6GB
```

---

## 12. Stage 8 — Build WinPE ARM64

WinPE is a stripped-down Windows environment that runs before the main OS. You boot it on the phone and use it to format the Windows partition and deploy `install.wim`.

Still on **Windows**.

### 12.1 Install Windows ADK

1. Go to: https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install
2. Download the **Windows ADK** installer
3. Run it — on the feature selection screen, check **Deployment Tools** only, uncheck everything else
4. On the same page, download the **Windows PE add-on for the ADK**
5. Run the WinPE add-on installer — accept defaults

### 12.2 Open the ADK Environment

Search Start for **"Deployment and Imaging Tools Environment"** and open it **as Administrator**. This is a special CMD window with ADK tools in the PATH.

### 12.3 Create WinPE ARM64 Working Directory

```cmd
copype arm64 C:\WinPE_arm64
```

This creates the WinPE folder structure with a base ARM64 boot image.

### 12.4 Mount the WinPE Image

```cmd
mkdir C:\WinPE_mount

dism /Mount-Image /ImageFile:"C:\WinPE_arm64\media\sources\boot.wim" /Index:1 /MountDir:"C:\WinPE_mount"
```

### 12.5 Add Required Packages to WinPE

```cmd
set PKGS=C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\arm64\WinPE_OCs

:: WMI support
dism /Image:C:\WinPE_mount /Add-Package /PackagePath:"%PKGS%\WinPE-WMI.cab"
dism /Image:C:\WinPE_mount /Add-Package /PackagePath:"%PKGS%\en-us\WinPE-WMI_en-us.cab"

:: .NET Framework
dism /Image:C:\WinPE_mount /Add-Package /PackagePath:"%PKGS%\WinPE-NetFX.cab"
dism /Image:C:\WinPE_mount /Add-Package /PackagePath:"%PKGS%\en-us\WinPE-NetFX_en-us.cab"

:: Scripting
dism /Image:C:\WinPE_mount /Add-Package /PackagePath:"%PKGS%\WinPE-Scripting.cab"
dism /Image:C:\WinPE_mount /Add-Package /PackagePath:"%PKGS%\en-us\WinPE-Scripting_en-us.cab"

:: DISM cmdlets (for driver injection from WinPE)
dism /Image:C:\WinPE_mount /Add-Package /PackagePath:"%PKGS%\WinPE-DismCmdlets.cab"
dism /Image:C:\WinPE_mount /Add-Package /PackagePath:"%PKGS%\en-us\WinPE-DismCmdlets_en-us.cab"

:: Storage and disk tools
dism /Image:C:\WinPE_mount /Add-Package /PackagePath:"%PKGS%\WinPE-StorageWMI.cab"
dism /Image:C:\WinPE_mount /Add-Package /PackagePath:"%PKGS%\en-us\WinPE-StorageWMI_en-us.cab"
```

### 12.6 Copy the Windows Image into WinPE Media

```cmd
mkdir C:\WinPE_arm64\media\win_sources

:: Copy install.wim — adjust path if yours is elsewhere
copy "C:\Users\%USERNAME%\Downloads\uup_download_windows\install.wim" ^
     "C:\WinPE_arm64\media\win_sources\install.wim"

:: Verify it copied
dir "C:\WinPE_arm64\media\win_sources\"
```

### 12.7 Unmount and Commit the WinPE Image

```cmd
dism /Unmount-Image /MountDir:C:\WinPE_mount /Commit
```

### 12.8 Create a Driver Folder in WinPE Media

You'll put your drivers here before writing to USB:
```cmd
mkdir C:\WinPE_arm64\media\drivers
```

We populate this in Stage 10.

### 12.9 Write WinPE to a USB Drive

Plug in a USB drive (8GB minimum). Open Disk Management (`diskmgmt.msc`) to find its drive letter (e.g., `E:`).

Back in the ADK environment:
```cmd
MakeWinPEMedia /UFD C:\WinPE_arm64 E:
```

> Replace `E:` with your actual drive letter. **This wipes the USB drive.**

When done, your USB drive is bootable WinPE ARM64 and contains the Windows `install.wim`.

---

## 13. Stage 9 — Deploy Windows from WinPE

Now you boot the phone directly into WinPE from the USB drive and deploy Windows to the `win` partition.

### 13.1 Switch Phone to UEFI Slot

Back in **WSL**:
```bash
# Make sure phone is plugged in and recognized
adb devices

adb reboot bootloader
```

Re-attach:
```powershell
usbipd attach --wsl --busid 2-3
```

```bash
# Switch to slot B (Renegade UEFI)
fastboot --set-active=b
fastboot reboot
```

The phone should boot into the Renegade UEFI menu.

### 13.2 Boot WinPE from USB

1. Plug your WinPE USB drive into the phone using a USB-C hub (or USB-C adapter if you have one)
2. In the Renegade UEFI screen:
   - Navigate to **Boot Manager**
   - Select the USB entry (it should appear as your USB drive or "UEFI: USB Drive")
3. WinPE loads — you'll get a CMD prompt

If the UEFI doesn't show USB boot, try holding Volume Down when the UEFI logo appears to enter the boot menu.

### 13.3 Map Partition Letters in WinPE

Inside the **WinPE CMD prompt on the phone**:

```cmd
:: Launch diskpart
diskpart

:: See all disks
list disk

:: Select the main storage (usually Disk 0)
select disk 0

:: See all partitions
list partition
```

Find your partitions. Look for:
- The ~512MB partition → that's `esp`
- The ~55GB partition → that's `win`

Note their partition numbers. Example output:
```
  Partition ###  Type              Size     Offset
  -------------  ----------------  -------  -------
  Partition 1    System            128 MB   ...
  ...
  Partition 30   Primary           407 GB   ...     ← userdata (shrunk)
  Partition 31   System            512 MB   ...     ← esp
  Partition 32   Primary           55 GB    ...     ← win
```

### 13.4 Format and Assign Drive Letters

```cmd
:: Assign drive letter to the Windows partition
select partition 32
format fs=ntfs quick label="Windows"
assign letter=W

:: Assign drive letter to the ESP
select partition 31
assign letter=S

:: Confirm assignments
list volume

exit
```

Verify:
```cmd
dir W:\
dir S:\
```

Both should exist without errors. `W:\` will be empty. `S:\` might have some existing EFI files from Android — that's fine, bcdboot will add to it.

### 13.5 Find the USB Drive Letter

```cmd
diskpart
list volume
exit
```

Find your USB drive in the list — it's the one labeled with the WinPE volume name (usually `WINPE` or similar). Note its drive letter (e.g., `D:` or `E:`).

### 13.6 Deploy Windows with DISM

```cmd
:: Replace D: with your USB drive letter if different
dism /Apply-Image ^
  /ImageFile:D:\win_sources\install.wim ^
  /Index:1 ^
  /ApplyDir:W:\
```

This takes **10–25 minutes**. You'll see a progress percentage. Don't disconnect anything.

When it finishes:
```cmd
:: Verify Windows files are present
dir W:\Windows\System32\ntoskrnl.exe
```

### 13.7 Apply Boot Files to the ESP

```cmd
bcdboot W:\Windows /s S: /f UEFI /l en-us
```

Expected output: `Boot files successfully created.`

### 13.8 Configure the BCD Store

```cmd
:: Enable test signing — required for unsigned ARM64 drivers
bcdedit /store S:\EFI\Microsoft\Boot\BCD /set {default} testsigning on

:: Disable integrity checks — allows unsigned kernel drivers
bcdedit /store S:\EFI\Microsoft\Boot\BCD /set {default} nointegritychecks on

:: Disable recovery mode (avoids getting stuck in recovery loops)
bcdedit /store S:\EFI\Microsoft\Boot\BCD /set {default} recoveryenabled no

:: Set data execution protection
bcdedit /store S:\EFI\Microsoft\Boot\BCD /set {default} nx OptIn

:: Verify the BCD looks correct
bcdedit /store S:\EFI\Microsoft\Boot\BCD /enum all
```

The output should show a Windows Boot Manager entry and a Windows Boot Loader entry pointing to `W:\Windows`.

### 13.9 Pre-Inject Drivers Before First Boot

If you have drivers on the USB (from Stage 10), inject them now while W: is still mounted:

```cmd
:: Inject all drivers from the drivers folder on USB
dism /Image:W:\ /Add-Driver /Driver:D:\drivers\ /Recurse /ForceUnsigned

:: Check what was injected
dism /Image:W:\ /Get-Drivers | findstr "Provider"
```

### 13.10 Prepare for First Boot

```cmd
:: Unmount volumes cleanly before rebooting
diskpart
select volume W
remove letter=W
select volume S
remove letter=S
exit
```

Reboot the phone:
```cmd
wpeutil reboot
```

---

## 14. Stage 10 — Collect and Inject Drivers

Drivers are what make hardware functional. Without them, most of the phone is invisible to Windows.

### 14.1 Get Drivers from Lenovo ThinkPad X13s Gen 1

The ThinkPad X13s uses the SC8280XP SoC, which is in the same SM8450 family as the Edge+'s SM8450. Many peripheral drivers are shared or compatible.

Back in **WSL**:
```bash
cd ~/woa-hiphi/drivers
mkdir -p x13s
cd x13s

# Search for the X13s driver pack on Lenovo's site
# URL format (verify current URL at support.lenovo.com):
X13S_URL="https://download.lenovo.com/pccbbs/mobiles/sc8280xp_arm64_driver_package.exe"

wget "$X13S_URL" -O x13s-drivers.exe

# Extract with 7zip
7z x x13s-drivers.exe -o./extracted/ -y
ls extracted/
```

Copy to Windows for use during WinPE injection:
```bash
WINUSER=$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d '\r')
mkdir -p "/mnt/c/Users/$WINUSER/Desktop/hiphi-drivers/x13s"
cp -r extracted/ "/mnt/c/Users/$WINUSER/Desktop/hiphi-drivers/x13s/"
echo "Copied to Windows Desktop"
```

### 14.2 Get WOA-Project Driver Pack

```bash
cd ~/woa-hiphi/drivers
git clone https://github.com/WOA-Project/Qualcomm-Reference-Drivers.git woa-project
ls woa-project/

# Copy to Windows
WINUSER=$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d '\r')
cp -r woa-project/ "/mnt/c/Users/$WINUSER/Desktop/hiphi-drivers/woa-project/"
```

### 14.3 Extract Drivers from a Windows ARM Device (If You Have One)

If you have access to a Windows ARM device (Surface Pro X, ThinkPad X13s, etc.):

```powershell
:: Run on the ARM Windows device
pnputil /export-driver * C:\extracted-drivers\
```

Copy the `C:\extracted-drivers\` folder to your USB drive.

### 14.4 Copy Drivers to Your WinPE USB

On Windows, copy all drivers to the USB drive:
```cmd
xcopy /E /I C:\Users\%USERNAME%\Desktop\hiphi-drivers E:\drivers
```

### 14.5 Inject Drivers from WinPE (If Doing It Live)

If Windows is already deployed and you're back in WinPE:

```cmd
:: Mount the Windows partition
diskpart
select disk 0
select partition 32
assign letter=W
exit

:: Inject drivers
dism /Image:W:\ /Add-Driver /Driver:D:\drivers\x13s\ /Recurse /ForceUnsigned
dism /Image:W:\ /Add-Driver /Driver:D:\drivers\woa-project\ /Recurse /ForceUnsigned

:: Confirm what got injected
dism /Image:W:\ /Get-Drivers

:: Unmount
diskpart
select volume W
remove letter=W
exit
```

### 14.6 Install Drivers After Windows Boots

Once Windows is running (Stage 11), from an elevated CMD:

```cmd
:: Install all drivers from a folder recursively
pnputil /add-driver C:\drivers\*.inf /install /subdirs

:: Or use Device Manager for individual devices
devmgmt.msc
```

For Device Manager: right-click any yellow `!` device → Update Driver → Browse my computer → point to `C:\drivers` → include subfolders.

---

## 15. Stage 11 — First Boot and OOBE

### 15.1 Reboot to Windows

From WSL:
```bash
adb reboot bootloader
```

Re-attach:
```powershell
usbipd attach --wsl --busid 2-3
```

```bash
# Switch to slot B (UEFI/Windows)
fastboot --set-active=b
fastboot reboot
```

### 15.2 Expected Boot Sequence

```
1. Motorola ABL (brief splash)
2. Renegade UEFI logo
3. Windows Boot Manager (no visible UI, just a second or two)
4. Windows logo + spinning dots
5. OOBE first-run screen
```

If it gets stuck anywhere, see the Troubleshooting section.

### 15.3 Required Hardware Before OOBE

You **cannot use the touchscreen** — there are no touch drivers. You need:

- **USB-C Hub** with at least 2 USB-A ports
- **USB Mouse** — mandatory
- **USB Keyboard** — mandatory for entering passwords/names
- Optional: USB-C Ethernet adapter (for more reliable network than Wi-Fi)

### 15.4 OOBE Walkthrough

1. **Country/Region** — Select with mouse, click Yes
2. **Keyboard layout** — Select, click Yes
3. **Second keyboard layout** — Click Skip
4. **Network** — Click **"I don't have internet"** at bottom left → Click **"Continue with limited setup"**
   > This bypasses the Microsoft account requirement and lets you create a local account.
5. **Name** — Type a username
6. **Password** — Can leave blank for now, click Next
7. **Security questions** — Skip or answer, click Next
8. **Privacy settings** — Turn off everything, click Accept
9. **Cortana** — Click Not now
10. **Desktop appears** ✅

### 15.5 Post-Install Configuration

First thing — open CMD as Administrator (Start → search "cmd" → right-click → Run as administrator):

```cmd
:: Re-enable test signing (OOBE sometimes resets this)
bcdedit /set testsigning on
bcdedit /set nointegritychecks on

:: Check current status
bcdedit /enum {current} | findstr -i "test\|integrity\|recovery"

:: Disable Windows Update to prevent it from overwriting your config
sc stop wuauserv
sc config wuauserv start=disabled
sc stop bits
sc config bits start=disabled
sc stop dosvc
sc config dosvc start=disabled

:: Open Device Manager to see what's missing
start devmgmt.msc
```

In Device Manager, yellow `!` marks are hardware without drivers. Common ones:
- Unknown Device × many — sensors, GPIO, etc.
- WCN6750 Wi-Fi — may appear as unknown
- Adreno 730 — will show as Microsoft Basic Display Adapter until a real driver exists

### 15.6 Enable ADB Remote Access from WSL

With USB mouse plugged into the hub, you can also use the USB port to connect to WSL:

1. On the phone: Settings → System → Developer Options → Enable USB Debugging
2. Unplug mouse hub, plug phone directly to PC
3. In PowerShell: `usbipd attach --wsl --busid 2-3`
4. In WSL:

```bash
adb devices
# Should show your Windows device

# Run PowerShell commands remotely
adb shell powershell "whoami"
adb shell powershell "Get-WmiObject Win32_OperatingSystem | Select-Object Caption,Version"

# Install a driver remotely
adb push ~/woa-hiphi/drivers/somedriver.inf /Windows/Temp/
adb shell powershell "pnputil /add-driver C:\Windows\Temp\somedriver.inf /install"
```

---

## 16. Dual Boot Reference

### Switch to Windows
```bash
# WSL
adb reboot bootloader
# PowerShell: usbipd attach --wsl --busid 2-3
fastboot --set-active=b
fastboot reboot
```

### Switch to Android
```bash
# WSL
adb reboot bootloader
# PowerShell: usbipd attach --wsl --busid 2-3
fastboot --set-active=a
fastboot reboot
```

### Check Which Slot Is Active
```bash
fastboot getvar current-slot
```

### Reflash UEFI After Changes
```bash
# After rebuilding edk2-msm
adb reboot bootloader
# PowerShell: usbipd attach --wsl --busid 2-3
fastboot --slot b flash boot \
  ~/woa-hiphi/edk2-msm/Build/MotoEdgePlus2022/RELEASE_CLANGPDB/FV/uefi.img
fastboot reboot
```

---

## 17. usbipd Quick Reference

You will run these commands constantly. Save these.

```powershell
# List all USB devices and their BUSIDs
usbipd list

# Bind a device (one-time per device)
usbipd bind --busid 2-3

# Attach to WSL (run after EVERY phone reboot or mode switch)
usbipd attach --wsl --busid 2-3

# Detach (hand back to Windows — needed for flashing from Windows native tools)
usbipd detach --busid 2-3

# If BUSID changes after replug
usbipd list
usbipd attach --wsl --busid NEW-BUSID
```

**In WSL after attaching:**
```bash
# Confirm device is visible
lsusb | grep -i "motorola\|android"

# Confirm ADB works
adb devices

# Confirm fastboot works (phone must be in fastboot mode)
fastboot devices
```

---

## 18. Troubleshooting

### Phone not detected in WSL

```bash
# Check if usbipd attached correctly
lsusb
# If phone is missing:
```
```powershell
# PowerShell — detach and reattach
usbipd detach --busid 2-3
Start-Sleep 3
usbipd attach --wsl --busid 2-3
```
```bash
# Try a different USB port on the PC
# Try a different cable
# Make sure the phone isn't in "Charging only" mode — pull down notification shade and change USB mode to "File Transfer"
```

### `adb devices` shows `unauthorized`

On the phone, a popup asks "Allow USB debugging?" — accept it. If no popup appears:
```bash
adb kill-server
adb start-server
adb devices
# Popup should appear now
```

### Fastboot shows `< waiting for device >`

Phone isn't in fastboot mode or not attached:
```bash
# On the phone: hold Power + Volume Down for 10 seconds to force fastboot
# Then re-attach:
```
```powershell
usbipd attach --wsl --busid 2-3
```
```bash
fastboot devices
```

### EDK2 build fails — `error: unknown type name '__int128'`

```bash
# Use GCC instead of Clang for this specific component:
sudo apt install -y gcc
# Some EDK2 versions need specific Clang; try clang-15:
sudo apt install -y clang-15 lld-15
sudo update-alternatives --install /usr/bin/clang clang /usr/bin/clang-15 200
./build.sh --device MotoEdgePlus2022 --release
```

### EDK2 build fails — `ModuleNotFoundError: No module named 'distutils'`

```bash
sudo apt install -y python3-distutils python3-setuptools
# Or install via pip:
pip3 install setuptools
./build.sh --device MotoEdgePlus2022 --release
```

### UEFI test boot — black screen

1. Check display config. Try swapping pixel depth:
```bash
sed -i 's/PcdMipiFrameBufferPixelBpp|32/PcdMipiFrameBufferPixelBpp|24/' \
  ~/woa-hiphi/edk2-msm/Platforms/SM8450/MotoEdgePlus2022/MotoEdgePlus2022.dsc
```
2. Rebuild and test again:
```bash
cd ~/woa-hiphi/edk2-msm
./build.sh --device MotoEdgePlus2022 --release && \
  fastboot boot Build/MotoEdgePlus2022/RELEASE_CLANGPDB/FV/uefi.img
```

### UEFI test boot — garbled screen / wrong colors

Resolution or framebuffer stride is wrong. Check kernel DTS for exact display geometry:
```bash
grep -r "mdss_dsi\|panel_resolution\|1080\|2400" \
  ~/woa-hiphi/hiphi-kernel/arch/arm64/boot/dts \
  --include="*.dtsi" | grep -v "^Binary" | head -20
```

### Windows BSOD on first boot

| Stop Code | Likely Cause | Fix |
|---|---|---|
| `INACCESSIBLE_BOOT_DEVICE` | BCD points to wrong partition, or NTFS format failed | Boot WinPE, run `diskpart` to verify W: has Windows files, re-run `bcdboot` |
| `SYSTEM_THREAD_EXCEPTION_NOT_HANDLED` | Incompatible driver injected | Boot WinPE, run `dism /Image:W:\ /Remove-Driver /Driver:INFNAME.inf` for recently added drivers |
| `PAGE_FAULT_IN_NONPAGED_AREA` | RAM base address wrong in UEFI DSC | Check `PcdRamSize` and memory layout |
| `MEMORY_MANAGEMENT` | UEFI memory map not passed correctly | Compare memory map with a working SM8450 UEFI device's DSC |
| Instant reboot, no BSOD | UEFI not initializing hardware correctly before handoff | Check UEFI serial UART output if possible |

### Windows stuck at spinning dots forever

```bash
# Reboot to WinPE, re-run bcdboot
# Also verify testsigning is on:
# Inside WinPE:
# bcdedit /store S:\EFI\Microsoft\Boot\BCD /enum {default}
# Look for "testsigning    Yes"
```

### Phone stuck in bootloop / won't reach fastboot

```bash
# Force fastboot by holding Power + Volume Down for 15 seconds
# Even if the screen is off, keep holding

# Once in fastboot, switch back to Android
fastboot --set-active=a
fastboot reboot
```

If Android slot A is also broken (you flashed something bad to it):
```bash
# Flash stock Android back
fastboot flash bootloader ~/woa-hiphi/firmware/retus/bootloader.img
fastboot reboot bootloader
# Re-attach
fastboot flash boot ~/woa-hiphi/firmware/retus/boot.img
fastboot --set-active=a
fastboot reboot
```

### `dism /Apply-Image` fails in WinPE

- Make sure `W:\` is formatted NTFS (not just created — actually formatted)
- Make sure you have enough space (`W:` partition must be larger than the `install.wim`)
- Try a different USB port on the phone's hub

### WinPE doesn't boot from USB

- Make sure UEFI is working (Renegade logo visible on boot)
- In Renegade UEFI, go to Boot Manager — USB drive should appear after a few seconds
- Try a different USB hub
- Make sure the USB drive was written with `MakeWinPEMedia /UFD` and not just copied

---

## 19. Driver Status Table

| Component | Hardware | Status | Notes |
|---|---|---|---|
| Display (basic) | Samsung pOLED | ✅ Works | Renegade linear framebuffer, 1080×2400, no HW accel |
| Display (GPU-accelerated) | Adreno 730 | ❌ No driver | Freedreno (Linux/Mesa) exists but no WoA port; phone runs software render |
| Touchscreen | Focaltech FT8726 | ❌ No driver | HID over I2C — needs a Windows HID miniport driver written from scratch |
| Physical buttons | Vol+/Vol−/Power | ⚠️ Partial | Power works; volume buttons may work via ACPI GPIO |
| USB-C (data/OTG) | USB 3.1 Gen 2 | ✅ Works | Hub, keyboard, mouse, storage all function |
| USB-C (power delivery) | PMIC | ⚠️ Partial | Basic slow charging only; TurboPower (fast charge) not working |
| Wi-Fi | WCN6750 | ⚠️ Partial | Driver from X13s pack may load but reliability varies; USB dongle recommended |
| Bluetooth | WCN6750 | ❌ No driver | Same chip as Wi-Fi but no Bluetooth WoA driver |
| NFC | NXP | ❌ No driver | |
| Audio (speaker/earpiece) | WCD9380 codec | ❌ No driver | Qualcomm audio DSP — no WoA support |
| Audio (USB-C) | USB audio class | ✅ Works | Standard USB audio via hub works fine |
| Cellular / 5G | Snapdragon X65 modem | ❌ Not planned | Windows doesn't use Qualcomm modem stack on phones |
| mmWave 5G UW | QTM527 | ❌ Not planned | Hardware is Android-only |
| Camera (all) | Various | ❌ No driver | Camera ISP entirely unsupported |
| Battery status | PMIC / BMS | ⚠️ Partial | Windows sees approximate % but no ACPI power management; no sleep/wake |
| Screen brightness | PMIC | ❌ No control | Fixed brightness only |
| Fingerprint sensor | Goodix | ❌ No driver | |
| Gyroscope / Accel / Compass | LSM6DSO / AK0991x | ❌ No driver | |
| Vibration motor | PMIC | ❌ No driver | |
| GPS | Snapdragon GPS | ❌ No driver | |

**Practical result:** You get a working Windows desktop with display, USB input, and possibly Wi-Fi. Everything else is missing. The phone is usable as a low-power ARM64 test machine with mouse and keyboard via USB-C hub.

---

## 20. Resources

| Resource | URL |
|---|---|
| Renegade Project (EDK2 for Snapdragon) | https://github.com/edk2-porting/edk2-msm |
| WOA-Project (drivers + tools) | https://github.com/WOA-Project |
| usbipd-win releases | https://github.com/dorssel/usbipd-win/releases |
| UUP Dump — ARM64 ISO builder | https://uupdump.net |
| Windows ADK download | https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install |
| SM8450 mainline Linux kernel | https://github.com/msm-mainline/linux |
| Motorola hiphi kernel source | https://github.com/MotorolaMobilityLLC/kernel-msm |
| postmarketOS hiphi wiki | https://wiki.postmarketos.org/wiki/Motorola_Edge+_(2022) |
| Motorola RETUS firmware mirror | https://mirrors.lolinet.com/firmware/motorola/hiphi/official/RETUS/ |
| Windows on Qualcomm Discord | Search "windows on qualcomm" on Discord |
| XDA hiphi device forum | https://forum.xda-developers.com/f/motorola-edge-2022.12577/ |
| TWRP device page | https://dl.twrp.me/hiphi/ |
| Lenovo X13s drivers (SM8450 ref) | https://support.lenovo.com (search ThinkPad X13s Gen 1) |
| Qualcomm ACPI / DSDT reference | SM8450 Developer Kit DSDT, available in Windows Insider builds |

---

## Contributing

If you make progress on any broken component — especially touch input or GPU:

1. **UEFI changes** → Open a PR at [edk2-porting/edk2-msm](https://github.com/edk2-porting/edk2-msm), tag it `hiphi`
2. **Driver findings** → Document in this repo and open a PR
3. **Discord** → Post findings in `#sm8450` or `#device-bring-up` channels in the WoA Discord

The postmarketOS `hiphi` port and this WoA port share the same kernel DTS, ACPI tables, and hardware topology knowledge. Progress on either helps the other.

---

*Motorola Edge+ 2022 UW (XT2201-4) · SM8450 / hiphi · WSL2 Full Guide · May 2026*
*Status: Pioneer bring-up · No complete port exists yet*

---

MIT License — Copyright (c) 2026 HacksYT23-2
