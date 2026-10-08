# SD Card GPT / Protective MBR Repair Guide
**Windows 10/11 + Ubuntu WSL2 + usbipd-win + GPT fdisk**

## AI Usage For Documentation
**I had ChatGPT throw this document together using everything I did to troubleshoot. I ran through all of this on three units successfully. Linux is out of my wheelhouse, so if you have any suggestions on how to improve upon this, by all means let me know.**

## Overview

This guide explains how to access an SD card through Ubuntu running inside Windows and repair a corrupted protective Master Boot Record (MBR) while preserving an otherwise valid GUID Partition Table (GPT).

It is intended for situations where:

- Windows cannot properly access an SD card's partitions.
- Ubuntu detects a valid GPT but reports a corrupted protective MBR.
- Existing partitions and files need to be preserved.

**WARNING:** Partition-table repairs modify disk metadata and carry a risk of data loss. Create a complete raw image backup before proceeding. Never assume that two SD cards have identical partition layouts or device identifiers.

---

# PART 1 — Requirements and Installation

## 1. System Requirements

- Windows 11, or a compatible Windows 10 installation
- Administrator access to Windows
- Internet connection for initial software installation
- USB SD card reader
- SD card containing the affected partitions
- Separate storage containing a complete raw image backup
  - You can take a raw image back up using HDD Raw Copy Tool (https://hddguru.com/software/HDD-Raw-Copy-Tool/)

Software required:

1. Windows Subsystem for Linux (WSL2)
2. Ubuntu Linux
3. usbipd-win
4. GPT fdisk (`gdisk` and `sgdisk`)

Microsoft's current usbipd-win instructions target Windows 11 with an x64 or ARM64 processor. Windows 10 support may require additional compatibility considerations.

## 2. Install WSL2 and Ubuntu

**Perform this section in Windows PowerShell as Administrator.**

If WSL is not installed, enter:

`wsl --install`

This enables the required Windows features and installs Ubuntu as the default Linux distribution.

Restart Windows if prompted.

If WSL is already installed but Ubuntu is missing, enter:

`wsl --install -d Ubuntu`

After installation, launch Ubuntu from the Windows Start menu.

On first launch, Ubuntu may ask you to create:

- A Linux username
- A Linux password

These credentials are separate from your Windows credentials.

To verify the installation, return to PowerShell and run:

`wsl --list --verbose`

Look for Ubuntu with VERSION `2`.

If Ubuntu reports VERSION `1`, you can convert it using:

`wsl --set-version Ubuntu 2`

To make WSL2 the default for future Linux distributions:

`wsl --set-default-version 2`

Update WSL if necessary:

`wsl --update`

**Note:** Do not reinstall or convert an existing Ubuntu installation if it already runs correctly under WSL2.

Official documentation:
https://learn.microsoft.com/en-us/windows/wsl/install

## 3. Install usbipd-win

usbipd-win allows Windows USB devices, including compatible SD card readers, to be passed through to WSL2.

**Install this on Windows, not inside Ubuntu.**

Open PowerShell as Administrator and run:

`winget install --interactive --exact dorssel.usbipd-win`

Follow the installation prompts.

The installer may require a system restart.

After installation, close PowerShell and open a fresh window so it recognizes the new executable.

Verify that usbipd-win is installed:

`winget list --id dorssel.usbipd-win`

Then test:

`usbipd list`

If PowerShell reports that `usbipd` is not recognized, use its full installation path:

`& "C:\Program Files\usbipd-win\usbipd.exe" list`

The ampersand (`&`) is PowerShell's call operator and must be included when executing the quoted path.

For the remainder of the guide, commands use the shorter `usbipd` form. If necessary, replace `usbipd` with:

`& "C:\Program Files\usbipd-win\usbipd.exe"`

Official documentation:
https://learn.microsoft.com/en-us/windows/wsl/connect-usb

## 4. Install GPT fdisk in Ubuntu

Launch Ubuntu.

Update its package information:

`sudo apt update`

Install GPT fdisk:

`sudo apt install gdisk`

Confirm installation if prompted.

The `gdisk` package provides the interactive GPT partition editor and the command-line `sgdisk` utility.

Verify both programs:

`command -v gdisk`

`command -v sgdisk`

Both should return executable paths.

GPT fdisk documentation:
https://www.rodsbooks.com/gdisk/

---

# PART 2 — Connect the SD Card to Ubuntu

## 5. Identify the USB Reader

Insert the SD card into the USB reader.

Launch Ubuntu and leave its terminal running.

Open Windows PowerShell as Administrator.

List attached USB devices:

`usbipd list`

Find the entry corresponding to the USB Mass Storage Device.

Record its BUSID.

**Example BUSID:** `4-1`

This identifier may change when connecting the reader to a different USB port.

## 6. Share and Attach the Reader

If the device's state is `Not shared`, run:

`usbipd bind --busid 4-1`

If already `Shared`, skip this command.

Attach it to Ubuntu:

`usbipd attach --wsl --busid 4-1`

Replace `4-1` with the actual BUSID.

If successful, Windows will temporarily release the USB reader to WSL2.

**Troubleshooting:** If you receive "There is no WSL 2 distribution running," open Ubuntu, leave it running, and retry the attach command.

## 7. Identify the SD Card in Ubuntu

Switch to Ubuntu.

Run:

`lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINTS`

Identify the SD card by capacity.

For example, a 16 GB SD card may appear as approximately 14.6 GiB.

Its Linux device name might be:

`/dev/sde`

**IMPORTANT:** This is only an example. Confirm the actual device name every time. Using the wrong device can damage another drive.

All subsequent commands use `/dev/sde` as the example device.

---

# PART 3 — Diagnose the GPT

## 8. Verify the Partition Table

Run:

`sudo sgdisk -v /dev/sde`

For the problem addressed by this guide, the output may include:

"Found valid GPT with corrupt MBR; using GPT and will write new protective MBR on save."

It may also conclude:

"No problems found."

This combination indicates that GPT fdisk can read the GPT structures even though the protective MBR needs attention.

A partition alignment warning alone does not necessarily prevent repair.

**STOP if:**

- No valid GPT exists.
- GPT checksums or partition entries are invalid.
- Partitions overlap unexpectedly.
- The card reports I/O errors.
- The card's partition layout does not match expectations.

Different problems require different recovery methods.

## 9. Inspect the Existing Partitions

Run:

`sudo gdisk /dev/sde`

At the interactive prompt, type:

`p`

Review the partition table.

Example from the original SD card:

| Partition | Size | Name |
|---|---|---|
| 1 | 56 MiB | boot |
| 2 | 6.6 GiB | root |
| 3 | 7.9 GiB | primary |

A different card may have a different layout.

To inspect the protective MBR, type:

`x`

Then:

`o`

A standard protective MBR generally contains a partition entry of type `0xEE`.

**Note:** GPT fdisk may display a protective MBR reconstructed in memory rather than the damaged on-disk version.

Return to the main menu:

`m`

If anything is unexpected, type `q` to exit without saving.

---

# PART 4 — Repair the Protective MBR

## 10. Perform the Repair

Before proceeding, confirm:

- A complete raw image backup exists on separate storage.
- The correct SD card is selected.
- The GPT is valid.
- All expected partitions are listed correctly.
- No partitions on the card are mounted.
- No additional unexpected errors have appeared.

At the `gdisk` main prompt, type:

`w`

When prompted to confirm the write operation, type:

`Y`

GPT fdisk will save the partition-table structures, including the reconstructed protective MBR.

**WARNING:** The operation modifies disk metadata. It does not intentionally erase existing files or reformat partitions, but it can still cause data loss if the wrong device or incorrect partition information is used.

Wait for completion.

## 11. Verify the Repair

Run:

`sudo sgdisk -v /dev/sde`

Confirm that the corrupted protective MBR warning no longer appears.

Then run:

`lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINTS`

The expected partitions should now be visible.

Inspect filesystem information:

`sudo blkid /dev/sde1 /dev/sde2 /dev/sde3`

Use the actual partition numbers for your card.

In the original case, these were:

- `/dev/sde1` — FAT32, BOOT
- `/dev/sde2` — ext4, root
- `/dev/sde3` — exFAT, EASYROMS

If the partition devices do not appear immediately, stop and investigate rather than repeating the write operation.

---

# PART 5 — Return the SD Card to Windows

## 12. Release the USB Reader

Ensure that no SD card partitions remain mounted in Ubuntu.

Return to Windows PowerShell.

Run:

`usbipd detach --busid 4-1`

Use the actual BUSID.

This releases the SD card reader from WSL2 and returns control to Windows.

## 13. Verify Access in Windows

Open File Explorer.

Check whether the Windows-compatible partitions, such as EASYROMS, are visible.

Verify that expected files and folders are present.

**Do not format the card if Windows prompts you to do so.**

If Windows offers automatic filesystem repair, postpone it until you have verified the condition of the files and backups.

## 14. Safely Remove the Card

- Close programs accessing the SD card.
- Use Windows Safely Remove Hardware.
- Wait for confirmation.
- Remove the SD card.

---

# PART 6 — Troubleshooting and Important Notes

## USB device does not appear in Ubuntu

Confirm that Ubuntu is running.

In PowerShell, run:

`usbipd list`

If the reader is shared but not attached, repeat:

`usbipd attach --wsl --busid 4-1`

Then check again in Ubuntu:

`lsblk -o NAME,SIZE,FSTYPE,TYPE`

## PowerShell cannot recognize usbipd

Close and reopen PowerShell.

If necessary, execute usbipd using the full installation path:

`& "C:\Program Files\usbipd-win\usbipd.exe" list`

If the executable is not found there, verify its actual installation directory.

## gdisk or sgdisk is missing

In Ubuntu, run:

`sudo apt update`

`sudo apt install gdisk`

## Windows reports that the device is in use

Confirm that all SD card partitions are unmounted in Ubuntu before detaching.

Do not force an unmount while files are being written.

## GPT repair versus filesystem repair

These are separate operations.

**GPT/protective MBR repair** addresses partition metadata.

**Filesystem repair** addresses problems inside FAT32, exFAT, ext4, or another filesystem.

Repairing the protective MBR does not automatically repair filesystem corruption.

In the original case, exFAT diagnostics identified eight corrupted file entries involving cluster allocation conflicts.

Automatic filesystem repair may truncate, discard, or alter affected files.

## Backup and Cloning Recommendations

A raw disk image includes:

- The protective MBR
- The GPT partition table
- Partition contents
- File data
- Existing filesystem corruption, if any

Keep an original backup image unchanged.

An additional image made after repair may be useful for documenting the current state.

However, do not assume that a repaired card with existing filesystem errors is a clean source for cloning other cards.

When repairing a second card, repeat the verification process on that card independently rather than assuming its partition structures are identical.

---

# Reference Documentation

Microsoft — Install WSL:
https://learn.microsoft.com/en-us/windows/wsl/install

Microsoft — Connect USB Devices to WSL:
https://learn.microsoft.com/en-us/windows/wsl/connect-usb

Ubuntu — GPT fdisk Package:
https://packages.ubuntu.com/noble/gdisk

GPT fdisk — Official Documentation:
https://www.rodsbooks.com/gdisk/

GPT fdisk — Repairing GPT Disks:
https://www.rodsbooks.com/gdisk/repairing.html

---

**End of Guide**

**Safety reminder:** Verify device identifiers, keep a separate raw image backup, and only write partition metadata when diagnostics confirm that the GPT is valid and the protective MBR is the actual problem.
