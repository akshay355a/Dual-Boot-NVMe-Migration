# Dual-Boot NVMe Migration Runbook
## Fedora and Windows – Migration, UEFI Boot Configuration, and Source Drive Erasure

**Revision:** 1 November 2026  
**Document type:** Anonymized technical reference  
**Privacy:** Machine-specific values intentionally removed

## 1. Purpose

This document records a migration from one NVMe SSD to another while retaining an existing Windows installation and moving Fedora to the destination SSD.

The objectives were:

1. Preserve the existing Windows installation.
2. Migrate Fedora without reinstalling it.
3. Configure Fedora and Windows to boot independently from the destination SSD.
4. Remove stale UEFI firmware entries.
5. Securely erase the source SSD before resale.
6. Verify that the system no longer depends on the source drive.

This is a reusable reference, not a universal copy-and-run script. Partition layouts and device identifiers must be discovered on the machine being serviced.

## 2. Privacy and Redaction

All identifying values have been replaced with placeholders.

Do not include the following in a public version:

- Computer manufacturer, model, CPU/GPU model, asset tag or firmware identifiers.
- SSD manufacturer, model, serial number, namespace and firmware revision.
- Filesystem UUIDs, PARTUUIDs and GPT disk GUIDs.
- EFI Boot#### IDs, hardware device paths and Windows BCD identifiers.
- Hostname, username, home-directory path, IP address, MAC address and Wi-Fi SSID.
- Product keys, BitLocker recovery keys, passwords, SSH keys, tokens or credentials.

Never publish unredacted output from `lsblk`, `nvme list`, `efibootmgr -v`, `dmidecode`, `lshw`, `ip addr` or Windows System Information.

## 3. Placeholder Convention

| Placeholder | Meaning |
|---|---|
| `<SOURCE_DISK>` | Source physical NVMe disk |
| `<SOURCE_NAMESPACE>` | Source NVMe namespace to erase |
| `<DEST_DISK>` | Destination physical NVMe disk |
| `<DEST_ESP_PARTITION>` | Destination EFI System Partition |
| `<DEST_BOOT_PARTITION>` | Destination `/boot` partition |
| `<DEST_BTRFS_PARTITION>` | Destination Btrfs partition |
| `<SOURCE_BTRFS_UUID>` | Source Btrfs filesystem UUID |
| `<DEST_BTRFS_UUID>` | Destination Btrfs filesystem UUID |
| `<DEST_BOOT_UUID>` | Destination `/boot` UUID |
| `<DEST_ESP_UUID>` | Destination EFI filesystem UUID |
| `<ESP_PARTITION_NUMBER>` | Destination ESP partition number |
| `<BOOT_ENTRY_ID>` | Firmware Boot#### identifier |

These are explanatory placeholders, not literal values. Replace them only after inspecting the actual machine.

## 4. Migration Overview

The system used UEFI firmware and GPT partitioning.

The conceptual destination layout was:

| Partition | Purpose | Filesystem |
|---|---|---|
| Existing Windows partitions | Preserve Windows installation and recovery environment | Existing Windows formats |
| Destination ESP | Fedora and Microsoft EFI boot files | FAT32 |
| Destination `/boot` | Kernels, initramfs and BLS entries | ext4 |
| Destination Btrfs | Fedora root and home subvolumes | Btrfs |

The Fedora root and home subvolumes were copied with `rsync`. The separate `/boot` filesystem was copied independently. Filesystem references and bootloader configuration were updated for the destination.

## 5. Pre-Migration Checklist

Before starting:

- Create and verify separate backups of important Windows and Fedora data.
- Prepare Windows recovery media and Fedora live/rescue media.
- Connect AC power.
- Check Windows Device Encryption or BitLocker status and retain the recovery key offline.
- Check Secure Boot status.
- Identify both physical SSDs by local model and serial information.
- Confirm the destination has enough space.
- Confirm which Windows partitions must be preserved.
- Keep copies of disk inventory and boot configuration locally.
- Do not erase the source until both operating systems boot independently from the destination.

A partition on the same SSD is not a backup.

## 6. Inventory

Boot a trusted Linux live/rescue environment and inspect the drives.

```bash
sudo nvme list

sudo lsblk -e7 -o \
NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,UUID,PARTUUID,MOUNTPOINTS

sudo blkid

sudo efibootmgr -v

findmnt -R /
```

Inspect Btrfs subvolumes:

```bash
sudo btrfs subvolume list <BTRFS_MOUNTPOINT>
```

Use these outputs to map physical drives, partitions, filesystem UUIDs and EFI entries.

**Do not assume a fixed relationship between a physical drive and a Linux device name.** Device numbering can change. Identify the correct disk before any destructive operation.

## 7. Mount the Filesystems

Create mount points:

```bash
sudo mkdir -p /mnt/source-root
sudo mkdir -p /mnt/source-home
sudo mkdir -p /mnt/source-boot
sudo mkdir -p /mnt/source-efi

sudo mkdir -p /mnt/dest-top
sudo mkdir -p /mnt/dest-root
sudo mkdir -p /mnt/dest-home
sudo mkdir -p /mnt/dest-root/boot
sudo mkdir -p /mnt/dest-root/boot/efi
```

Inspect the source Btrfs subvolumes:

```bash
sudo mount -o subvolid=5 \
UUID=<SOURCE_BTRFS_UUID> /mnt/source-root

sudo btrfs subvolume list /mnt/source-root
sudo umount /mnt/source-root
```

Mount the source root and home subvolumes:

```bash
sudo mount -o subvol=root \
UUID=<SOURCE_BTRFS_UUID> /mnt/source-root

sudo mount -o subvol=home \
UUID=<SOURCE_BTRFS_UUID> /mnt/source-home
```

Mount the destination Btrfs filesystem and create subvolumes only if they do not already exist:

```bash
sudo mount -o subvolid=5 \
/dev/<DEST_BTRFS_PARTITION> /mnt/dest-top

sudo btrfs subvolume create /mnt/dest-top/root
sudo btrfs subvolume create /mnt/dest-top/home

sudo umount /mnt/dest-top

sudo mount -o subvol=root \
/dev/<DEST_BTRFS_PARTITION> /mnt/dest-root

sudo mount -o subvol=home \
/dev/<DEST_BTRFS_PARTITION> /mnt/dest-home
```

Keep destination `/boot` and ESP mount points unmounted during the initial root/home copy. This avoids writing root-subvolume directory metadata onto a mounted boot filesystem.

Verify:

```bash
findmnt /mnt/source-root /mnt/source-home
findmnt /mnt/dest-root /mnt/dest-home
```

## 8. Copy Fedora Data

The migration used an offline file-level copy with metadata-preserving `rsync` options.

Copy root:

```bash
sudo rsync -aHAXS --numeric-ids --info=progress2 \
/mnt/source-root/ /mnt/dest-root/
```

Copy home:

```bash
sudo rsync -aHAXS --numeric-ids --info=progress2 \
/mnt/source-home/ /mnt/dest-home/
```

Mount the separate `/boot` and ESP filesystems after root and home have been copied:

```bash
sudo mount /dev/<SOURCE_BOOT_PARTITION> /mnt/source-boot

sudo mount /dev/<DEST_BOOT_PARTITION> \
/mnt/dest-root/boot

sudo mount /dev/<SOURCE_ESP_PARTITION> \
/mnt/source-efi

sudo mount /dev/<DEST_ESP_PARTITION> \
/mnt/dest-root/boot/efi

findmnt /mnt/source-boot /mnt/dest-root/boot
findmnt /mnt/source-efi /mnt/dest-root/boot/efi
```

Copy `/boot`:

```bash
sudo rsync -aHAX --numeric-ids --info=progress2 \
/mnt/source-boot/ /mnt/dest-root/boot/
```

Important `rsync` options:

- `-a`: preserve common file metadata and directory structure.
- `-H`: preserve hard links.
- `-A`: preserve ACLs.
- `-X`: preserve extended attributes.
- `-S`: handle sparse files.
- `--numeric-ids`: preserve numeric ownership.
- `--info=progress2`: display transfer progress.

Do not add `--delete` without a specific reason. It can delete destination files that do not exist on the source.

Do not perform an inconsistent live copy while the source system is actively writing files.

## 9. Update `/etc/fstab`

The copied system initially refers to filesystem identifiers from the source drive.

Back up the copied fstab file:

```bash
sudo cp /mnt/dest-root/etc/fstab \
/mnt/dest-root/etc/fstab.pre-migration

sudoedit /mnt/dest-root/etc/fstab
```

Example logical layout:

```fstab
UUID=<DEST_BTRFS_UUID>  /          btrfs  subvol=root,compress=zstd:1  0 0
UUID=<DEST_BOOT_UUID>   /boot      ext4   defaults                    1 2
UUID=<DEST_ESP_UUID>    /boot/efi  vfat   umask=0077,shortname=winnt  0 2
UUID=<DEST_BTRFS_UUID>  /home      btrfs  subvol=home,compress=zstd:1 0 0
```

Preserve any other required mount points, swap configuration and filesystem options from the original system.

Validate:

```bash
sudo findmnt --verify \
--tab-file /mnt/dest-root/etc/fstab
```

## 10. Update Fedora Boot Entries

Fedora commonly stores Boot Loader Specification entries under:

```text
/boot/loader/entries/
```

Inspect root UUID references:

```bash
grep -RIn "root=UUID=" \
/mnt/dest-root/boot/loader/entries

sudo grub2-editenv \
/mnt/dest-root/boot/grub2/grubenv list
```

Update only references confirmed to point to the old root filesystem. Preserve existing kernel parameters.

Do not globally replace every UUID across the filesystem. UUIDs can also identify unrelated filesystems and swap devices.

## 11. Populate the Destination EFI Partition

Copy only the Fedora EFI directory from the source ESP when those boot files are known to be valid:

```bash
sudo mkdir -p \
/mnt/dest-root/boot/efi/EFI/fedora

sudo rsync -aHAX \
/mnt/source-efi/EFI/fedora/ \
/mnt/dest-root/boot/efi/EFI/fedora/
```

Do not blindly copy the entire source ESP over the destination ESP. That could overwrite Windows or vendor boot files.

**Important:** The EFI-side Fedora configuration file at:

```text
/boot/efi/EFI/fedora/grub.cfg
```

must locate the destination `/boot` filesystem. If copied from the source ESP, it may still refer to the old `/boot` UUID.

Inspect it and compare the search UUID with the destination `/boot` UUID:

```bash
cat /mnt/dest-root/boot/efi/EFI/fedora/grub.cfg
```

If it is missing or incorrect, repair it using Fedora's supported shim/GRUB package workflow for the installed release.

For modern Fedora UEFI installations, do not use `grub2-install` as a generic repair step. Use the distribution-managed shim and GRUB packages.

## 12. Rebuild GRUB Configuration

Bind the required virtual filesystems:

```bash
sudo mount --rbind /dev /mnt/dest-root/dev
sudo mount --make-rslave /mnt/dest-root/dev

sudo mount --rbind /proc /mnt/dest-root/proc
sudo mount --make-rslave /mnt/dest-root/proc

sudo mount --rbind /sys /mnt/dest-root/sys
sudo mount --make-rslave /mnt/dest-root/sys

sudo mount --rbind /run /mnt/dest-root/run
sudo mount --make-rslave /mnt/dest-root/run

sudo chroot /mnt/dest-root
```

Inside the chroot:

```bash
findmnt / /boot /boot/efi

cat /etc/fstab

grep -RIn "root=UUID=" /boot/loader/entries

cat /boot/efi/EFI/fedora/grub.cfg

grub2-mkconfig -o /boot/grub2/grub.cfg
```

Review the generated output for Fedora and Windows discovery.

If the source disk is still connected, the generated menu may include its old installation. Remove those entries only after the source is erased or disconnected and the configuration is regenerated.

## 13. Recreate Windows Boot Files

Boot Windows or Windows Recovery Environment.

Assign a temporary drive letter to the verified destination ESP using DiskPart:

```cmd
diskpart
list disk
select disk <DESTINATION_DISK_NUMBER>
list partition
select partition <DESTINATION_ESP_PARTITION_NUMBER>
assign letter=<TEMP_ESP_LETTER>
exit
```

Verify that the selected volume is the destination FAT32 EFI System Partition.

Run BCDBoot from an elevated command prompt:

```cmd
bcdboot <WINDOWS_VOLUME>\Windows /s <TEMP_ESP_LETTER>: /f UEFI /v
```

The Windows volume may not be `C:` in Recovery Environment. Identify the correct Windows installation before running the command.

Verify that this file exists on the destination ESP:

```text
<ESP_LETTER>:\EFI\Microsoft\Boot\bootmgfw.efi
```

BCDBoot copies Windows boot-environment files from the selected Windows installation to the specified system partition.

## 14. Create and Verify UEFI Entries

After verifying that the EFI files exist, create destination entries.

Fedora example:

```bash
sudo efibootmgr --create \
  --disk /dev/<DEST_DISK> \
  --part <ESP_PARTITION_NUMBER> \
  --label "Fedora (Destination)" \
  --loader '\EFI\fedora\shimx64.efi'
```

Windows example:

```bash
sudo efibootmgr --create \
  --disk /dev/<DEST_DISK> \
  --part <ESP_PARTITION_NUMBER> \
  --label "Windows Boot Manager (Destination)" \
  --loader '\EFI\Microsoft\Boot\bootmgfw.efi'
```

Inspect:

```bash
sudo efibootmgr -v
```

Verify that each entry points to the destination ESP by matching its GPT partition GUID/PARTUUID. Do not rely on display labels alone.

## 15. Boot-Test Before Erasing

1. Reboot and select the Fedora loader on the destination ESP from the firmware one-time boot menu.
2. After Fedora starts, verify the mounted filesystems:

```bash
findmnt -no SOURCE,FSTYPE,UUID / /boot /boot/efi
sudo efibootmgr -v
```

3. Reboot and select the destination Windows Boot Manager.
4. Confirm that Windows reaches the desktop and the expected data volume is intact.
5. Repeat tests after any boot-entry or GRUB change.

**Do not erase the source until both Fedora and Windows have booted independently from the destination.**

## 16. Remove Stale UEFI Entries

Inspect:

```bash
sudo efibootmgr -v
```

Map each entry to its partition GUID and EFI loader path.

Keep the verified destination Fedora entry, destination Windows entry, useful destination fallback entry and USB boot entry.

Remove only confirmed stale entries:

```bash
sudo efibootmgr -b <STALE_BOOT_ENTRY_ID> -B
```

After every deletion, inspect the list again.

For duplicate Windows entries, retain the one that has already booted successfully. Remove a duplicate only after verifying that both point to the same destination EFI loader.

After the source is erased or removed, regenerate GRUB:

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

Confirm the old installation is no longer listed.

## 17. Erase the Source NVMe SSD

**Destructive operation:** The following command permanently erases user data on the selected NVMe namespace.

First inspect the source drive:

```bash
sudo nvme list

sudo nvme id-ctrl /dev/<SOURCE_CONTROLLER> -H

sudo lsblk -o \
NAME,SIZE,MODEL,SERIAL,FSTYPE,UUID,PARTUUID,MOUNTPOINTS \
/dev/<SOURCE_DISK>
```

Verify:

- The namespace belongs to the source SSD, not the destination.
- No source partitions are mounted.
- No process is using the source drive.
- Both operating systems have successfully booted from the destination.
- The destination is the current working system.
- The selected device path still matches the source drive inventory.

The command used for User Data Erase is:

```bash
sudo nvme format /dev/<SOURCE_NAMESPACE> --ses=1
```

Read the warning carefully. Cancel immediately if the namespace named by the command is not the intended source.

### NVMe Secure Erase Settings

| SES | Meaning |
|---|---|
| `0` | No secure erase requested |
| `1` | User Data Erase |
| `2` | Cryptographic Erase |

SES=1 requests User Data Erase. The controller is instructed to erase user data, but the post-erase contents are indeterminate. The controller may use cryptographic erase if all user data is encrypted.

SES=1 must not be described as proof that every physical NAND cell has been overwritten. It also does not prove that advanced laboratory recovery is mathematically impossible.

For regulated or highly sensitive data, use the manufacturer's documented sanitization method and the applicable media-destruction policy.

## 18. Verify the Erased Drive

Inspect the namespace and GPT:

```bash
sudo nvme list

sudo sgdisk --print /dev/<SOURCE_DISK>

sudo lsblk -f /dev/<SOURCE_DISK>
```

In the documented migration, the namespace reported zero user-data usage and GPT inspection showed no partition entries.

Linux may temporarily retain stale partition nodes or filesystem signatures in its running device map. If the output looks inconsistent, refresh the partition map when safe:

```bash
sudo partprobe /dev/<SOURCE_DISK>
```

A controlled reboot can also refresh the device state.

Do not create a new partition table or filesystem simply to make the drive appear empty. Leave it unpartitioned for resale.

## 19. Physical Removal and Final Acceptance

1. Shut down completely.
2. Disconnect power and follow the laptop manufacturer's service procedure.
3. Follow appropriate ESD precautions.
4. Remove only the erased source SSD identified locally.
5. Reassemble the laptop.
6. Boot Fedora from the destination firmware entry.
7. Verify `/`, `/boot` and `/boot/efi` are mounted from the destination.
8. Boot Windows from the destination firmware entry.
9. Confirm that both operating systems work with the source SSD physically absent.

The migration is fully independent only after both operating systems pass this post-removal cold-boot test.

## 20. Troubleshooting

| Symptom | Likely cause | Response |
|---|---|---|
| Fedora enters emergency mode | Old UUID in fstab or missing mount | Check destination UUIDs and fstab from live media |
| Old Fedora appears in GRUB | Source still connected or stale custom entry | Remove source dependency and regenerate GRUB |
| Windows missing from GRUB | Windows EFI files or os-prober entry missing | Verify destination Microsoft EFI files and Windows UEFI entry |
| Windows entry fails | Wrong ESP or missing boot files | Match PARTUUID and run BCDBoot against verified ESP |
| Duplicate firmware entries | Old entries or duplicate creation | Match entries by partition GUID and loader path |
| Old partitions still appear after format | Stale kernel partition map | Inspect GPT; refresh safely or reboot |
| Fedora only boots with source installed | Destination EFI, fstab or BLS still depends on source | Repair destination boot chain before erasing |
| Secure Boot failure | Signed bootloader chain is inconsistent | Use Fedora-supported shim/GRUB packages |

## 21. Lessons Learned

- Filesystem UUIDs and GPT PARTUUIDs are more reliable than Linux device numbering.
- Keep a private before-and-after inventory.
- Test both operating systems independently before source erasure.
- Verify the Fedora EFI-side GRUB stub points to the new `/boot` filesystem.
- Do not overwrite the entire destination ESP with a blind copy.
- Inspect NVMe controller capabilities before choosing an erase method.
- Format NVM SES=1, Sanitize and Cryptographic Erase are different operations.
- A stale partition node is not proof that the secure erase failed.
- Delete firmware entries only after verifying their actual partition GUIDs and loader paths.
- Keep recovery media and offline backups until the final post-removal boot test succeeds.

## 22. Completion Record

| Checkpoint | Result |
|---|---|
| Fedora root and home migration | Completed |
| Separate `/boot` migration | Completed |
| Destination fstab and BLS updates | Completed |
| Destination Fedora EFI files | Present and boot-tested |
| Destination Windows EFI files | Created and boot-tested |
| Destination UEFI entries | Created and verified |
| Fedora boot from destination | Passed |
| Windows boot from destination | Passed |
| Source NVMe Format NVM SES=1 | Completed successfully |
| Namespace/GPT inspection | Zero reported usage; no GPT partition entries |
| Stale source UEFI entries | Removed |
| Duplicate Windows entry | Removed after verification |
| Physical source removal and final cold boot | Perform and record locally |

## 23. References

1. Linux NVMe CLI – `nvme-format` manual: https://github.com/linux-nvme/nvme-cli/blob/master/Documentation/nvme-format.1
2. Microsoft Learn – BCDBoot command-line options: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/bcdboot-command-line-options-techref-di
3. Microsoft Learn – NVMe Secure Erase Settings: https://learn.microsoft.com/en-us/windows/win32/api/nvme/ne-nvme-nvme_secure_erase_settings
4. Fedora Project – GRUB 2: https://fedoraproject.org/wiki/GRUB_2
5. Fedora Project – Btrfs: https://fedoraproject.org/wiki/Btrfs

## Appendix A. Before Publishing

- Review the text for real models, serials, UUIDs, GUIDs, disk paths, namespace paths, usernames, hostnames, network addresses and product keys.
- Inspect screenshots separately; text redaction does not redact image attachments.
- Remove document author metadata and tracked changes.
- Never upload raw terminal logs or firmware screenshots without sanitizing them.
- Keep the unredacted private service record separate from the public copy.

**Privacy note:** This runbook uses generic placeholders. Review any future additions before sharing them publicly.
