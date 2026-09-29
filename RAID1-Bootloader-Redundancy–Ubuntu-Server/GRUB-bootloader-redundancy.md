# RAID1 Bootloader Redundancy – Ubuntu Server

## Overview

This SOP describes how to configure **GRUB bootloader redundancy** on an Ubuntu production server using:

* Software RAID1 (`mdadm`)
* GPT partition table
* Legacy BIOS boot mode
* GRUB 2.x
* Two physical disks

The objective is to ensure that **either physical disk can independently boot the server if the other disk fails**.

> **Production Warning:** Partition-table and bootloader changes should be performed during an approved maintenance window. Always maintain console/iDRAC/iLO access before making bootloader changes.

---

## 1. Example Environment

### Physical disks

```text
/dev/sda  → 120 GB SSD
/dev/sdb  → 120 GB SSD
```

### RAID configuration

```text
md0 → RAID1 → swap
md1 → RAID1 → /
```

### Boot mode

```text
Legacy BIOS
```

### Target configuration

```text
/dev/sda
├── BIOS Boot Partition
├── RAID1 member
└── RAID1 member

/dev/sdb
├── BIOS Boot Partition
├── RAID1 member
└── RAID1 member
```

Both disks must contain GRUB boot code.

---

# 2. Pre-Checks

## 2.1 Check Physical Disk Health

```bash
smartctl -H /dev/sda
smartctl -H /dev/sdb
```

Expected:

```text
SMART overall-health self-assessment test result: PASSED
```

For detailed health information:

```bash
smartctl -a /dev/sda
smartctl -a /dev/sdb
```

---

## 2.2 Check RAID Status

```bash
cat /proc/mdstat
```

Expected:

```text
md1 : active raid1 sda2[1] sdb3[0]
      [2/2] [UU]

md0 : active raid1 sda1[1] sdb2[0]
      [2/2] [UU]
```

`[UU]` means both RAID members are active.

Also verify:

```bash
mdadm --detail /dev/md0
mdadm --detail /dev/md1
```

Expected:

```text
State : clean
Active Devices : 2
Failed Devices : 0
```

---

# 3. Verify Boot Mode

```bash
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "Legacy BIOS"
```

For this procedure, the expected result is:

```text
Legacy BIOS
```

---

# 4. Check Current Disk Layout

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,PARTTYPE,PARTFLAGS,PARTLABEL,MOUNTPOINTS
```

Check GPT:

```bash
parted /dev/sda print
parted /dev/sdb print
```

For GPT/Legacy BIOS, look for a partition with:

```text
bios_grub
```

Example:

```text
Number  Start   End     Size    Name       Flags

1       ...     ...     ...     BIOS Boot  bios_grub
```

---

# 5. Verify Existing GRUB

Check the first sector of each disk:

```bash
dd if=/dev/sda bs=512 count=1 2>/dev/null | strings
```

```bash
dd if=/dev/sdb bs=512 count=1 2>/dev/null | strings
```

Look for:

```text
GRUB
Geom
Hard Disk
Read
Error
```

If GRUB is present on only one disk, bootloader redundancy is incomplete.

---

# 6. Backup GPT Partition Tables

Before modifying any partition table:

```bash
sgdisk --backup=/root/sda-gpt-backup-$(date +%F-%H%M).bin /dev/sda
```

```bash
sgdisk --backup=/root/sdb-gpt-backup-$(date +%F-%H%M).bin /dev/sdb
```

Verify:

```bash
ls -lh /root/*gpt-backup-*.bin
```

### Important

Copy the backup files to another system or storage location if possible.

A partition-table backup stored on the same physical disk does not protect against complete disk failure.

---

# 7. Validate GPT

```bash
sgdisk -v /dev/sda
```

```bash
sgdisk -v /dev/sdb
```

Expected:

```text
No problems found.
```

---

# 8. Check for Free Space

Before creating a BIOS Boot partition:

```bash
parted /dev/sda unit MiB print free
```

For GPT/Legacy BIOS, approximately 1 MiB of free space can be used for a BIOS Boot partition.

**Do not resize or move existing RAID partitions unless specifically planned and approved.**

---

# 9. Create BIOS Boot Partition

If suitable free space exists before the first RAID partition:

```bash
sgdisk --new=3:34:2047 \
       --typecode=3:ef02 \
       --change-name=3:'BIOS Boot' \
       /dev/sda
```

Verify:

```bash
sgdisk -p /dev/sda
```

Expected:

```text
Number  Start  End   Size       Code  Name

3       34     2047  ~1 MiB     EF02  BIOS Boot
```

Also verify:

```bash
parted /dev/sda print
```

Expected:

```text
BIOS Boot    bios_grub
```

---

# 10. Install GRUB on the Second Disk

Install GRUB to `/dev/sda`:

```bash
grub-install --target=i386-pc --recheck /dev/sda
```

Expected:

```text
Installing for i386-pc platform.
Installation finished. No error reported.
```

Then update the GRUB configuration:

```bash
update-grub
```

---

# 11. Verify GRUB on Both Disks

Check `/dev/sda`:

```bash
dd if=/dev/sda bs=512 count=1 2>/dev/null | strings
```

Check `/dev/sdb`:

```bash
dd if=/dev/sdb bs=512 count=1 2>/dev/null | strings
```

Both disks should contain:

```text
GRUB
Geom
Hard Disk
Read
Error
```

This confirms GRUB boot code is present on both physical disks.

---

# 12. Verify RAID After the Change

```bash
cat /proc/mdstat
```

Expected:

```text
[UU]
```

for all RAID1 arrays.

Then:

```bash
mdadm --detail /dev/md0
mdadm --detail /dev/md1
```

Verify:

```text
State : clean
Active Devices : 2
Failed Devices : 0
```

---

# 13. Final Configuration

The desired configuration is:

```text
                    Physical Disks
                 ┌────────┴────────┐
                 │                 │
              /dev/sda          /dev/sdb
                 │                 │
              GRUB ✅           GRUB ✅
                 │                 │
           BIOS Boot          BIOS Boot
                 │                 │
              RAID1             RAID1
                 └────────┬────────┘
                          │
                    mdadm RAID1
                          │
                       OS/Data
```

Therefore:

```text
GRUB on /dev/sda → YES
GRUB on /dev/sdb → YES
RAID1 redundancy → YES
```

---

# 14. Disk Failure Scenarios

## Scenario 1 – `/dev/sda` fails

```text
/dev/sda → FAILED
/dev/sdb → WORKING
```

Expected:

* RAID continues in degraded mode.
* Data remains available from `/dev/sdb`.
* GRUB is available on `/dev/sdb`.
* Server can boot from `/dev/sdb`.

---

## Scenario 2 – `/dev/sdb` fails

```text
/dev/sdb → FAILED
/dev/sda → WORKING
```

Expected:

* RAID continues in degraded mode.
* Data remains available from `/dev/sda`.
* GRUB is available on `/dev/sda`.
* Server can boot from `/dev/sda`.

---

# 15. Controlled Boot-Failover Test

The strongest verification is an actual boot test during an approved maintenance window.

### Test 1

1. Shut down the server normally.
2. Disconnect/remove `/dev/sda` using the server's supported hardware procedure.
3. Boot the server.
4. Confirm that it boots successfully from `/dev/sdb`.
5. Verify RAID status.

### Test 2

1. Shut down the server.
2. Restore `/dev/sda`.
3. Verify RAID rebuild/synchronization.
4. After synchronization, shut down again.
5. Disconnect/remove `/dev/sdb`.
6. Boot the server.
7. Confirm that it boots successfully from `/dev/sda`.

> **Do not physically remove or disconnect a disk from a production server without following the server manufacturer's supported hot-swap or maintenance procedure.**

---

# 16. Post-Failure Recovery

After replacing a failed disk:

1. Partition the replacement disk with the required RAID and BIOS Boot partitions.
2. Add the RAID partitions to the existing arrays.
3. Monitor RAID rebuild.
4. Install GRUB on the replacement disk.
5. Verify RAID status.
6. Verify GRUB on both disks.

Check RAID rebuild:

```bash
cat /proc/mdstat
```

Wait until the RAID returns to:

```text
[UU]
```

---

# 17. Quick Health Check

Use these commands for routine verification:

```bash
smartctl -H /dev/sda
smartctl -H /dev/sdb

cat /proc/mdstat

mdadm --detail /dev/md0
mdadm --detail /dev/md1

lsblk -o NAME,SIZE,TYPE,FSTYPE,PARTTYPE,PARTFLAGS,PARTLABEL,MOUNTPOINTS

dd if=/dev/sda bs=512 count=1 2>/dev/null | strings

dd if=/dev/sdb bs=512 count=1 2>/dev/null | strings
```

### Success Criteria

```text
Disk health              → PASSED
RAID status              → [UU]
GRUB on /dev/sda        → YES
GRUB on /dev/sdb        → YES
BIOS boot partition      → BOTH DISKS
Single-disk failure      → RAID + BOOT FAILOVER
```

---

## Important Production Notes

* Always maintain remote console access such as **iDRAC/iLO/IPMI** before bootloader work.
* Take a GPT partition-table backup before partition changes.
* Do not resize RAID partitions unnecessarily.
* Do not run `partprobe` blindly on an active production RAID system after changing the partition table.
* Perform physical disk failover testing only during an approved maintenance window.
* Verify RAID synchronization before considering a replacement disk fully recovered.
* Keep a copy of the GPT backups outside the affected server.
