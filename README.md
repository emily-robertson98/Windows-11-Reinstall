# Windows 11 Reinstall on HP Pavilion x360 14-dw1010wm — NVMe Driver Troubleshooting

## Overview

This documents a full Windows 11 reinstall on an HP Pavilion x360 14-dw1010wm that required manual NVMe driver injection to resolve an Intel VMD controller conflict. The process involved WinRE command-line troubleshooting, diskpart partition management, manual driver loading via drvload, and a Windows 10 intermediate install before upgrading to Windows 11.

**Device:** HP Pavilion x360 14-dw1010wm  
**Processor:** Intel Core i5-1135G7 (11th Gen)  
**Storage:** Samsung MZVLQ256HBJD NVMe PCIe M.2 SSD (256GB)  
**OS Reinstalled:** Windows 10 Home → upgraded to Windows 11  

---

## The Problem

The laptop would not boot into Windows. When entering the Windows Recovery Environment (WinRE), the following issues were encountered:

- `Reset this PC` option was missing from the WinRE menu
- `diskpart` showed no fixed disks
- `dir C:\` returned "system cannot find path specified"
- No WiFi interface was available in WinRE
- WiFi driver was not loaded in the WinPE environment

The root cause was the **Intel VMD (Volume Management Device) controller**, which is enabled by default on 11th Gen Intel platforms. WinRE and the Windows installer do not include the VMD storage controller driver, making the NVMe SSD completely invisible to diskpart and the installer.

---

## Tools Used

- Windows 10 Media Creation Tool
- Rufus (bootable USB creation)
- Intel RST VMD Driver (sp134298 from HP Support)
- Samsung NVMe Driver (extracted .inf file)
- diskpart, drvload, dism (Windows PE utilities)

---

## Step-by-Step Process

### 1. Entering WinRE
- Booted into WinRE by allowing Windows to fail to boot three consecutive times
- Accessed Command Prompt via Troubleshoot → Advanced Options

### 2. Diagnosing the Drive Issue
Ran the following commands to diagnose:

```cmd
diskpart
list disk
```
Result: No fixed disks found.

```cmd
dism /image:C:\ /get-packages
```
Result: Responded successfully — confirmed Windows partition was partially accessible at some point but became inaccessible.

```cmd
wmic logicaldisk get name
bcdedit
```
Used to identify available drive letters and boot entries.

### 3. Network Troubleshooting (Dead End)
Attempted to connect to WiFi for cloud recovery:

```cmd
net start wlansvc
netsh wlan show networks
wpeutil InitializeNetwork
```
Result: No wireless interface on the system — WiFi driver not loaded in WinPE. USB tethering via Android phone also failed. Network recovery was not possible.

### 4. Creating a Bootable USB
On a second PC:
- Downloaded Windows 10 Media Creation Tool from microsoft.com
- Created bootable USB (GPT, UEFI)
- Also used Rufus as an alternative USB creation tool

### 5. Identifying the NVMe Driver Issue
Booted from USB and reached the drive selection screen — no drives were listed. Used Load Driver to attempt installing storage drivers:

- Downloaded Intel RST VMD driver (sp134298) from hp.com/support
- Extracted .exe to locate F6 folder containing .inf files
- Attempted to load via Load Driver GUI — driver showed but failed to install
- Attempted drvload via command prompt — loaded successfully but drive still not visible

Also downloaded Samsung NVMe driver for the specific SSD model (MZVLQ256HBJD) and extracted the .inf file.

### 6. Identifying the Correct Driver Load Order
Key discovery: **Intel VMD driver must be loaded before the Samsung NVMe driver.**

In the installer command prompt (Shift+F10):

```cmd
drvload D:\Drivers\VMD\[Intel VMD .inf]
```
Wait a few seconds, then:
```cmd
drvload D:\Drivers\Samsung\[Samsung NVMe .inf]
rescan
```

After loading in this order, diskpart finally showed:

```
Disk 0 — USB Drive
Disk 1 — 238GB (Samsung NVMe SSD)
```

### 7. Preparing the Drive

```cmd
diskpart
select disk 1
clean
convert gpt
exit
```

### 8. Resolving Installation Errors

Several errors were encountered during installation:

**Error 1: "Windows cannot be installed on this disk" (USB device error)**  
Cause: Windows misidentified the NVMe SSD as a USB-connected device due to missing VMD driver.  
Fix: Load Intel VMD driver before proceeding.

**Error 2: BSOD — INACCESSIBLE_BOOT_DEVICE during installation**  
Cause: VMD driver not persisting through installation reboots.  
Fix: Switched from Windows 11 to Windows 10 installer, which handled the driver reboots more reliably.

**Error 3: "We couldn't create a new partition or locate an existing one"**  
Cause: Two USB drives plugged in simultaneously confused the installer.  
Fix: Loaded drivers from second USB, then unplugged it before proceeding with installation.

### 9. Successful Installation
- Booted from Windows 10 USB with only the installation USB plugged in
- Loaded Intel VMD driver then Samsung NVMe driver via drvload
- Unplugged driver USB after loading
- Selected Disk 1, deleted existing partitions, selected unallocated space
- Windows 10 installed successfully

### 10. Upgrading to Windows 11
After Windows 10 was fully updated:
- Visited microsoft.com/software-download/windows11
- Downloaded Windows 11 Installation Assistant
- Ran the assistant — upgraded to Windows 11 with no issues

---

## Key Takeaways

- **Intel VMD controller** on 11th Gen Intel platforms hides NVMe drives from WinPE/WinRE by default — this is a known issue on HP Pavilions and many other OEM laptops
- **Driver load order matters** — Intel VMD must load before the NVMe drive-specific driver
- **drvload** is more reliable than the Load Driver GUI in the Windows installer for this use case
- **Multiple USB drives** plugged in simultaneously can cause partition errors during Windows installation
- **Windows 10 installer** handles VMD driver persistence through reboots better than Windows 11 installer
- BIOS did not expose VMD/AHCI toggle options on this HP Pavilion model, making driver injection the only viable path

---

## Commands Reference

```cmd
# Check for visible disks
diskpart
list disk
rescan

# Load drivers manually in WinPE
drvload X:\path\to\driver.inf

# Prepare disk for installation
select disk 1
clean
convert gpt

# Find .inf files on a drive
dir D:\ /s *.inf

# Network initialization in WinPE
wpeutil InitializeNetwork

# Bypass Microsoft account during Windows setup
oobe\bypassnro
```

---

## Environment

- **Date:** June 2026
- **Skill Level:** Intermediate — hands-on WinPE command-line troubleshooting
- **Time to Resolution:** ~6 hours
- **Outcome:** Successful Windows 11 reinstall on previously unbootable laptop
