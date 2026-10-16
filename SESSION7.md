# SESSION 7 — Ansible Service Masking, Debugging, Partitions, LVM, Filesystems, and Cron

> **Preservation note:** The full original `SESSION 7.txt` content is included verbatim at the end of this file. Every original command line and terminal output is retained. The explanations below are additional study notes and safety clarifications; they do not replace the source material.

## Contents
1. Masking and unmasking services
2. `debug` module
3. `uri` module note
4. `parted` module
5. `filesystem` module
6. `lvg` module
7. `lvol` module
8. Mounting an LVM logical volume
9. `cron` module
10. Quick revision and safety reminders
11. Original notes — preserved verbatim

---

## 1. Masking and unmasking a service

### What is service masking?
On a systemd-based Linux system, masking a service links its unit file to `/dev/null`, preventing the service from being started normally. Disabling a service only prevents automatic startup at boot; a disabled service can usually still be started manually. A masked service must be unmasked before it can be started.

### Mask a service using Linux
```bash
systemctl mask httpd
```
Unmask it:
```bash
systemctl unmask httpd
```

### Manage masking with Ansible
Mask `httpd` on the hosts in the `suman` inventory pattern:
```bash
ansible suman -m systemd -a 'name=httpd masked=true'
```
The source notes then stop the service:
```bash
ansible suman -m systemd -a 'name=httpd state=stopped'
```
Unmask it again:
```bash
ansible suman -m systemd -a 'name=httpd masked=false'
```

**Important:** Masking and service runtime state are different settings. Masking does not necessarily stop a service that is already running; stop it separately if required. Likewise, unmasking does not automatically start the service. To unmask and start it, use an appropriate task with `masked=false` and `state=started` (with privilege escalation when required). Service administration usually requires root privileges, so use `-b` if privilege escalation is not configured elsewhere.

## 2. `debug` module

### Purpose
The `debug` module displays messages and variable values during Ansible execution. It is useful for troubleshooting playbooks and checking values without changing the managed system.

### Example from the notes
```bash
ansible liu -m debug -a 'msg="hello bhai"'
```
This displays `hello bhai` for each host in the `liu` pattern.

Common uses include:
- printing a simple message with `msg=`;
- displaying a variable with `var=`;
- checking gathered facts and task results while debugging a playbook.

A debug task is informational; it normally does not change the managed node.

## 3. `uri` module note

The original notes contain:
```bash
ansible all -m url -a 'url=google.com'
```

This is preserved as written in the source section. The Ansible HTTP/HTTPS request module is named **`uri`**, not `url`. A typical valid example uses a complete URL and the `uri` module, for example:
```bash
ansible all -m uri -a 'url=https://www.google.com'
```
A network request can fail because of DNS, connectivity, TLS certificate validation, proxy configuration, or HTTP response status. Use a full URL scheme such as `https://` and inspect the returned error before troubleshooting.

## 4. `parted` module

### Purpose
The `community.general.parted` module manages disk partition tables and partitions. It can create and remove partitions and define their start and end positions.

### Install the collection
```bash
ansible-galaxy collection install community.general
```

### Create a 10 GiB partition numbered 1
Source command:
```bash
ansible suman -m parted -a 'device=/dev/sda number=1 part_end=10GiB state=present'
```
Parameter meanings:
- `device=/dev/sda`: disk to modify.
- `number=1`: partition number.
- `part_end=10GiB`: end position of the partition.
- `state=present`: ensure the partition exists.

The source output shows a 20 GiB `/dev/sda` and a first partition of about 10 GiB. It also includes a collection compatibility warning. Check Ansible and collection versions if module warnings or failures occur.

### Create a 5 GiB partition numbered 2
```bash
ansible suman -m parted -a 'device=/dev/sda number=2 part_start=10GiB part_end=15GiB state=present'
```
This requests partition 2 from 10 GiB to 15 GiB, a 5 GiB span.

### Create an extended partition in an MBR/msdos partition table
```bash
ansible suman -m parted -a 'device=/dev/sda number=3 part_start=15GiB part_end=19GiB part_type=extended state=present'
```
The source uses an `msdos` partition table and requests an extended partition spanning 15–19 GiB. Extended partitions are a feature of MBR/msdos partitioning, not GPT. The sample `lsblk` output reports `sda3` as approximately `1K`, which is normal for an extended partition container in `lsblk`; it represents the container for logical partitions rather than a regular data partition.

### Verify partitions
```bash
ansible suman -m command -a 'lsblk'
```
`lsblk` lists disks and partitions, their sizes, types, and mount points.

### Remove partition 3
```bash
ansible suman -m parted -a 'device=/dev/sda number=3 state=absent'
```
This removes the partition-table entry for partition 3. Confirm the correct device and partition number first; partition changes can make data inaccessible. Removing a partition entry is not the same as securely erasing all data from the disk.

**Critical disk safety:** The examples target `/dev/sda` on lab VMs. Device names and existing partitions differ across machines. Inspect `lsblk`, confirm the target disk, back up important data, and avoid running partitioning commands against a production or system disk unless you are certain of the intended result.

## 5. `filesystem` module

### Purpose
The filesystem module creates a filesystem such as XFS or ext4 on a block device or partition.

### Example: create XFS on `/dev/sda1`
```bash
ansible suman -m filesystem -a 'device=/dev/sda1 fstype=xfs'
```
- `device=/dev/sda1`: target partition or block device.
- `fstype=xfs`: filesystem type to create.

The source also shows:
```bash
ansible suman -m filesystem -a 'device=/dev/volgrp/logvol fstype=ext4'
```
This creates an ext4 filesystem on the logical volume path shown in the notes.

**Warning:** Creating a filesystem can overwrite existing filesystem metadata and make previous data inaccessible. Verify the device or logical volume before executing the command. Do not run filesystem-creation tasks on a device that already contains data you need to keep.

Verify filesystem type and UUID:
```bash
ansible suman -m command -a 'lsblk -f'
```
Verify mounted filesystems and usage:
```bash
ansible suman -m command -a 'df -Th'
```

## 6. `lvg` module

### Purpose
The LVM volume-group module manages LVM volume groups (VGs). A typical LVM structure is:

1. **Physical volume (PV):** a disk or partition initialized for LVM.
2. **Volume group (VG):** a storage pool made from one or more PVs.
3. **Logical volume (LV):** a virtual block device allocated from a VG.
4. **Filesystem:** such as XFS or ext4, created on the LV.
5. **Mount point:** directory where the filesystem is mounted.

### Create a volume group from `/dev/sda2`
```bash
ansible suman -m lvg -a 'pvs=/dev/sda2 vg=volgrp state=present'
```
- `pvs=/dev/sda2`: physical volume device(s) to include.
- `vg=volgrp`: volume-group name.
- `state=present`: ensure the VG exists.

Verify:
```bash
ansible suman -m command -a 'vgs'
```

### Add `/dev/sda1` to the existing volume group
```bash
ansible suman -m lvg -a 'pvs=/dev/sda2,/dev/sda1 vg=volgrp state=present'
```
This requests a VG named `volgrp` using both partitions. The sample output shows the volume group growing from about 5 GiB to about 15 GiB.

**Important source/output detail:** The source output later shows `/dev/sda1` and `/dev/sda2` both associated with LVM metadata. This is expected after both devices are added to the VG. Do not assume a device is unused simply because it appears in an example.

### Remove a volume group
```bash
ansible suman -m lvg -a 'vg=volgrp state=absent'
```
This requests removal of the volume group. The source does this after the earlier VG/LV exercises. Removing a VG can destroy access to logical volumes in it. Ensure dependent logical volumes are no longer needed and that data is backed up before removing a VG.

## 7. `lvol` module

### Purpose
The `community.general.lvol` module creates, resizes, and removes LVM logical volumes. It is used after the volume group has been created.

### Example storage layout
The notes describe:
- PV: `/dev/sdb1`
- VG: `vgdata`
- LV: `lvdata`, sized at 2 GiB

The actual command examples use VG `volgrp` and LV names `logvol` and `test`.

### Create a 4 GiB logical volume
```bash
ansible suman -m lvol -a 'vg=volgrp lv=logvol size=4G state=present'
```
- `vg=volgrp`: volume group.
- `lv=logvol`: logical-volume name.
- `size=4G`: requested size.
- `state=present`: ensure it exists.

Verify:
```bash
ansible suman -m command -a 'lvs'
```

### Increase the LV size by 2 GiB
```bash
ansible suman -m lvol -a 'vg=volgrp lv=logvol size=+2G state=present'
```
The `+2G` notation requests an increase of 2 GiB. The sample output shows `logvol` growing from 4 GiB to 6 GiB.

### Allocate all free space to another LV
```bash
ansible suman -m lvol -a 'vg=volgrp lv=test size=100%FREE state=present'
```
This requests an LV named `test` using all currently free space in the volume group. The sample output shows approximately 9 GiB allocated.

### Remove an LV
```bash
ansible suman -m lvol -a 'vg=volgrp lv=test state=absent force=true'
```
This removes `test`; `force=true` permits forced removal. Removing a logical volume destroys access to the data stored on it. Confirm the LV name and back up data first.

**Version note:** The source examples use `-m lvg` and `-m lvol` after installing `community.general`. The exact collection/module availability depends on the Ansible environment; consult `ansible-doc community.general.lvol` and `ansible-doc community.general.lvg` if Ansible cannot find a module.

## 8. Create a filesystem on an LV and mount it

The notes format `/dev/volgrp/logvol` as ext4, then mount it at `/opt`.

Create the filesystem:
```bash
ansible suman -m filesystem -a 'device=/dev/volgrp/logvol fstype=ext4'
```

Inspect block devices and filesystems:
```bash
ansible suman -m command -a 'lsblk -f'
```

Inspect mounted filesystem usage:
```bash
ansible suman -m command -a 'df -Th'
```

Mount the LV at `/opt` and configure a persistent mount:
```bash
ansible suman -m mount -a 'path=/opt src=/dev/volgrp/logvol fstype=ext4 opts=defaults state=mounted'
```
- `path=/opt`: mount point.
- `src=/dev/volgrp/logvol`: logical volume to mount.
- `fstype=ext4`: filesystem type.
- `opts=defaults`: standard mount options.
- `state=mounted`: mount now and ensure a persistent entry is present in `/etc/fstab`.

Verify with:
```bash
ansible suman -m command -a 'df -Th'
```

**Technical caution:** In the source `lsblk -f` output, both `/dev/sda1` and `/dev/sda2` appear under the same logical volume after the VG is formed. That is the expected LVM relationship, but the filesystem belongs on the logical volume—not separately on each PV. Formatting the LV is the appropriate operation for the example.

## 9. `cron` module

### Purpose
The Ansible `cron` module manages scheduled jobs in a user's crontab. It can create, update, and remove cron entries.

### Example from the notes
```bash
ansible suman -m cron -a 'name=suman weekday=5 minute=45 hour=14 user=root job="echo suman bhai > /var/log/messages"'
```

Parameter meanings:
- `name=suman`: descriptive identifier for the managed cron job.
- `weekday=5`: day of week value 5 (cron implementations commonly interpret this as Friday; check the target system's cron documentation).
- `minute=45`: run at minute 45.
- `hour=14`: run at 14:45 (2:45 PM).
- `user=root`: install the job in root's crontab.
- `job="echo suman bhai > /var/log/messages"`: command to execute.

The schedule shown is 14:45 on the configured weekday. The job redirects the text into `/var/log/messages`, replacing that file's contents if the shell redirection succeeds. This is a potentially destructive and unusual way to write to a system log. For a safer learning exercise, use a dedicated test file, for example `/tmp/cron-test.log`, and make sure the job's permissions and output are appropriate.

The task uses `user=root`; this generally requires Ansible privilege escalation or suitable permissions. Verify the installed cron entry using the appropriate crontab command on the target system. The original command is preserved unchanged below.

## 10. Quick revision table

| Module | Main use | Key parameters/examples |
|---|---|---|
| `systemd` | Mask/unmask and control services | `masked=true`, `masked=false`, `state=stopped` |
| `debug` | Print messages or variables | `msg=`, `var=` |
| `uri` | Make HTTP/HTTPS requests | `url=https://...` |
| `community.general.parted` | Create/remove disk partitions | `device=`, `number=`, `part_start=`, `part_end=`, `state=` |
| `filesystem` | Create filesystems | `device=`, `fstype=xfs` or `ext4` |
| `lvg` | Manage volume groups | `pvs=`, `vg=`, `state=` |
| `lvol` | Manage logical volumes | `vg=`, `lv=`, `size=`, `state=` |
| `mount` | Mount filesystems and manage `/etc/fstab` | `path=`, `src=`, `fstype=`, `state=mounted` |
| `cron` | Schedule recurring jobs | `name=`, `weekday=`, `hour=`, `minute=`, `job=`, `user=` |

## RHCE safety checklist

- Confirm the inventory group and hosts before running commands.
- Inspect `lsblk` and confirm the target disk before partitioning or formatting.
- Treat `mkfs`, filesystem creation, LV deletion, VG deletion, and partition removal as destructive.
- Use privilege escalation where needed.
- Verify the result with `lsblk`, `lsblk -f`, `vgs`, `lvs`, `df -Th`, or service status commands.
- The original source text below is preserved exactly, including commands that the explanatory sections flag for review.

---

# Original SESSION 7 notes — preserved verbatim


```text
SESSION 7


masking in ansible ->

frist we learn what is masking without ansible 
---
when masking a service, the symbolic links are moved and then point to /dev/null. when you simply disable a service, it can still be started manually . when you mask a service, it cannot be started manually . in other words, masking a service renders the service permanently unusable until it,s unmasked

TO mask a service called httpd or any service

- systemctl mask httpd 
then we can not start or enabled it 
if we want to start the service again we need to unmask it frist 
- systemctl unmask httpd

now do it in manage notes by using ansible 
my httpd service is now enabled and start -

then run -  ansible suman -m systemd -a 'name=httpd masked=true'
it will Mask the httpd service but if we check status it can stil enabled start 
but when we stop the survice by using -
ansible suman -m systemd -a 'name=httpd state=stopped'

then again try to enabled and start it we can not start it 
we need to unmask it to start and enabled the service again by using 

ansible suman -m systemd -a 'name=httpd masked=false'


-----------------------------------------------
===================================================
===================================================
--------------------------------------------------
debug module ------------>

in Ansible, the debug module is used to display messages, variable values, and the results of tasks while running a playbook. It is mainly used for troubleshooting and checking whether your automation is working correctly.


----- 
display message hello bhai 


suman1@ansible-server:~$ ansible liu -m debug -a 'msg="hello bhai"'
192.168.105.4 | SUCCESS => {
    "msg": "hello bhai"
}
192.168.105.5 | SUCCESS => {
    "msg": "hello bhai"
}
192.168.105.6 | SUCCESS => {
    "msg": "hello bhai"
}

---------------------------------
=================================
=================================
---------------------------------

uri module ----->

ansible all -m url -a 'url=google.com'

-------------------------------------------------
=================================================
=================================================
parted module

in Ansible, the community.general.parted module is used to manage disk partitions on Linux servers, such as RHEL 9 and RHEL 10.
It allows you to create, delete, and resize partitions on a disk using Ansible automation.

download it - suman1@ansible-server:~$ ansible-galaxy collection install community.general


make 1 10gb partition on suman group nodes 

suman1@ansible-server:~$ ansible suman -m parted -a 'device=/dev/sda number=1 part_end=10GiB state=present'
[WARNING]: Collection community.general does not support Ansible version 2.16.16
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "disk": {
        "dev": "/dev/sda",
        "logical_block": 512,
        "model": "ATA VMware Virtual S",
        "physical_block": 512,
        "size": 20971520.0,
        "table": "msdos",
        "unit": "kib"
    },
    "partitions": [
        {
            "begin": 1024.0,
            "end": 10485760.0,
            "flags": [],
            "fstype": "",
            "name": "",
            "num": 1,
            "size": 10484736.0,
            "unit": "kib"
        }
    ],
    "script": [
        "unit",
        "KiB",
        "mklabel",
        "msdos",
        "mkpart",
        "primary",
        "0%",
        "10GiB"
    ]
}
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "disk": {
        "dev": "/dev/sda",
        "logical_block": 512,
        "model": "ATA VMware Virtual S",
        "physical_block": 512,
        "size": 20971520.0,
        "table": "msdos",
        "unit": "kib"
    },
    "partitions": [
        {
            "begin": 1024.0,
            "end": 10485760.0,
            "flags": [],
            "fstype": "",
            "name": "",
            "num": 1,
            "size": 10484736.0,
            "unit": "kib"
        }
    ],
    "script": [
        "unit",
        "KiB",
        "mklabel",
        "msdos",
        "mkpart",
        "primary",
        "0%",
        "10GiB"
    ]
}
suman1@ansible-server:~$ ansible suman -m command -a 'lsblk'
node1 | CHANGED | rc=0 >>
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    0   20G  0 disk 
└─sda1          8:1    0   10G  0 part 
sr0            11:0    1 10.3G  0 rom  /run/media/suman/RHEL-10-2-BaseOS-x86_64
nvme0n1       259:0    0   40G  0 disk 
├─nvme0n1p1   259:1    0  600M  0 part /boot/efi
├─nvme0n1p2   259:2    0    2G  0 part /boot
└─nvme0n1p3   259:3    0 37.4G  0 part 
  ├─rhel-root 253:0    0 33.5G  0 lvm  /
  └─rhel-swap 253:1    0  3.9G  0 lvm  [SWAP]
node2 | CHANGED | rc=0 >>
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    0   20G  0 disk 
└─sda1          8:1    0   10G  0 part 
sr0            11:0    1 10.3G  0 rom  /run/media/suman/RHEL-10-2-BaseOS-x86_64
nvme0n1       259:0    0   40G  0 disk 
├─nvme0n1p1   259:1    0  600M  0 part /boot/efi
├─nvme0n1p2   259:2    0    2G  0 part /boot
└─nvme0n1p3   259:3    0 37.4G  0 part 
  ├─rhel-root 253:0    0 33.5G  0 lvm  /
  └─rhel-swap 253:1    0  3.9G  0 lvm  [SWAP]
suman1@ansible-server:~$ 


-----------------------------------------------------------
create a partition 5gb with partition number 2 
suman1@ansible-server:~$ ansible suman -m parted -a 'device=/dev/sda number=2 part_start=10GiB part_end=15GiB state=present'
[WARNING]: Collection community.general does not support Ansible version 2.16.16
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "disk": {
        "dev": "/dev/sda",
        "logical_block": 512,
        "model": "ATA VMware Virtual S",
        "physical_block": 512,
        "size": 20971520.0,
        "table": "msdos",
        "unit": "kib"
    },
    "partitions": [
        {
            "begin": 1024.0,
            "end": 10485760.0,
            "flags": [],
            "fstype": "",
            "name": "",
            "num": 1,
            "size": 10484736.0,
            "unit": "kib"
        },
        {
            "begin": 10485760.0,
            "end": 15728640.0,
            "flags": [],
            "fstype": "",
            "name": "",
            "num": 2,
            "size": 5242880.0,
            "unit": "kib"
        }
    ],
    "script": [
        "unit",
        "KiB",
        "mkpart",
        "primary",
        "10GiB",
        "15GiB"
    ]
}
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "disk": {
        "dev": "/dev/sda",
        "logical_block": 512,
        "model": "ATA VMware Virtual S",
        "physical_block": 512,
        "size": 20971520.0,
        "table": "msdos",
        "unit": "kib"
    },
    "partitions": [
        {
            "begin": 1024.0,
            "end": 10485760.0,
            "flags": [],
            "fstype": "",
            "name": "",
            "num": 1,
            "size": 10484736.0,
            "unit": "kib"
        },
        {
            "begin": 10485760.0,
            "end": 15728640.0,
            "flags": [],
            "fstype": "",
            "name": "",
            "num": 2,
            "size": 5242880.0,
            "unit": "kib"
        }
    ],
    "script": [
        "unit",
        "KiB",
        "mkpart",
        "primary",
        "10GiB",
        "15GiB"
    ]
}
suman1@ansible-server:~$ ansible suman -m command -a 'lsblk'
node1 | CHANGED | rc=0 >>
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    0   20G  0 disk 
├─sda1          8:1    0   10G  0 part 
└─sda2          8:2    0    5G  0 part 
sr0            11:0    1 10.3G  0 rom  /run/media/suman/RHEL-10-2-BaseOS-x86_64
nvme0n1       259:0    0   40G  0 disk 
├─nvme0n1p1   259:1    0  600M  0 part /boot/efi
├─nvme0n1p2   259:2    0    2G  0 part /boot
└─nvme0n1p3   259:3    0 37.4G  0 part 
  ├─rhel-root 253:0    0 33.5G  0 lvm  /
  └─rhel-swap 253:1    0  3.9G  0 lvm  [SWAP]
node2 | CHANGED | rc=0 >>
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    0   20G  0 disk 
├─sda1          8:1    0   10G  0 part 
└─sda2          8:2    0    5G  0 part 
sr0            11:0    1 10.3G  0 rom  /run/media/suman/RHEL-10-2-BaseOS-x86_64
nvme0n1       259:0    0   40G  0 disk 
├─nvme0n1p1   259:1    0  600M  0 part /boot/efi
├─nvme0n1p2   259:2    0    2G  0 part /boot
└─nvme0n1p3   259:3    0 37.4G  0 part 
  ├─rhel-root 253:0    0 33.5G  0 lvm  /
  └─rhel-swap 253:1    0  3.9G  0 lvm  [SWAP]
suman1@ansible-server:~$ 

---------------------------------------------------------------

create a 4gb extended partition 

suman1@ansible-server:~$ ansible suman -m parted -a 'device=/dev/sda number=3 part_start=15GiB part_end=19GiB part_type=extended state=present'
[WARNING]: Collection community.general does not support Ansible version 2.16.16
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "disk": {
        "dev": "/dev/sda",
        "logical_block": 512,
        "model": "ATA VMware Virtual S",
        "physical_block": 512,
        "size": 20971520.0,
        "table": "msdos",
        "unit": "kib"
    },
    "partitions": [
        {
            "begin": 1024.0,
            "end": 10485760.0,
            "flags": [],
            "fstype": "",
            "name": "",
            "num": 1,
            "size": 10484736.0,
            "unit": "kib"
        },
        {
            "begin": 10485760.0,
            "end": 15728640.0,
            "flags": [],
            "fstype": "",
            "name": "",
            "num": 2,
            "size": 5242880.0,
            "unit": "kib"
        },
        {
            "begin": 15728640.0,
            "end": 19922944.0,
            "flags": [
                "lba"
            ],
            "fstype": "",
            "name": "",
            "num": 3,
            "size": 4194304.0,
            "unit": "kib"
        }
    ],
    "script": [
        "unit",
        "KiB",
        "mkpart",
        "extended",
        "15GiB",
        "19GiB"
    ]
}
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "disk": {
        "dev": "/dev/sda",
        "logical_block": 512,
        "model": "ATA VMware Virtual S",
        "physical_block": 512,
        "size": 20971520.0,
        "table": "msdos",
        "unit": "kib"
    },
    "partitions": [
        {
            "begin": 1024.0,
            "end": 10485760.0,
            "flags": [],
            "fstype": "",
            "name": "",
            "num": 1,
            "size": 10484736.0,
            "unit": "kib"
        },
        {
            "begin": 10485760.0,
            "end": 15728640.0,
            "flags": [],
            "fstype": "",
            "name": "",
            "num": 2,
            "size": 5242880.0,
            "unit": "kib"
        },
        {
            "begin": 15728640.0,
            "end": 19922944.0,
            "flags": [
                "lba"
            ],
            "fstype": "",
            "name": "",
            "num": 3,
            "size": 4194304.0,
            "unit": "kib"
        }
    ],
    "script": [
        "unit",
        "KiB",
        "mkpart",
        "extended",
        "15GiB",
        "19GiB"
    ]
}
suman1@ansible-server:~$ ansible suman -m command -a 'lsblk'
node1 | CHANGED | rc=0 >>
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    0   20G  0 disk 
├─sda1          8:1    0   10G  0 part 
├─sda2          8:2    0    5G  0 part 
└─sda3          8:3    0    1K  0 part 
sr0            11:0    1 10.3G  0 rom  /run/media/suman/RHEL-10-2-BaseOS-x86_64
nvme0n1       259:0    0   40G  0 disk 
├─nvme0n1p1   259:1    0  600M  0 part /boot/efi
├─nvme0n1p2   259:2    0    2G  0 part /boot
└─nvme0n1p3   259:3    0 37.4G  0 part 
  ├─rhel-root 253:0    0 33.5G  0 lvm  /
  └─rhel-swap 253:1    0  3.9G  0 lvm  [SWAP]
node2 | CHANGED | rc=0 >>
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    0   20G  0 disk 
├─sda1          8:1    0   10G  0 part 
├─sda2          8:2    0    5G  0 part 
└─sda3          8:3    0    1K  0 part 
sr0            11:0    1 10.3G  0 rom  /run/media/suman/RHEL-10-2-BaseOS-x86_64
nvme0n1       259:0    0   40G  0 disk 
├─nvme0n1p1   259:1    0  600M  0 part /boot/efi
├─nvme0n1p2   259:2    0    2G  0 part /boot
└─nvme0n1p3   259:3    0 37.4G  0 part 
  ├─rhel-root 253:0    0 33.5G  0 lvm  /
  └─rhel-swap 253:1    0  3.9G  0 lvm  [SWAP]
suman1@ansible-server:~$ 

------------------------------------------------------------
remove sda3 -


---
suman1@ansible-server:~$ ansible suman -m parted -a 'device=/dev/sda number=3 state=absent'
[WARNING]: Collection community.general does not support Ansible version 2.16.16
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "disk": {
        "dev": "/dev/sda",
        "logical_block": 512,
        "model": "ATA VMware Virtual S",
        "physical_block": 512,
        "size": 20971520.0,
        "table": "msdos",
        "unit": "kib"
    },
    "partitions": [
        {
            "begin": 1024.0,
            "end": 10485760.0,
            "flags": [],
            "fstype": "",
            "name": "",
            "num": 1,
            "size": 10484736.0,
            "unit": "kib"
        },
        {
            "begin": 10485760.0,
            "end": 15728640.0,
            "flags": [],
            "fstype": "",
            "name": "",
            "num": 2,
            "size": 5242880.0,
            "unit": "kib"
        }
    ],
    "script": [
        "rm",
        "3"
    ]
}
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "disk": {
        "dev": "/dev/sda",
        "logical_block": 512,
        "model": "ATA VMware Virtual S",
        "physical_block": 512,
        "size": 20971520.0,
        "table": "msdos",
        "unit": "kib"
    },
    "partitions": [
        {
            "begin": 1024.0,
            "end": 10485760.0,
            "flags": [],
            "fstype": "",
            "name": "",
            "num": 1,
            "size": 10484736.0,
            "unit": "kib"
        },
        {
            "begin": 10485760.0,
            "end": 15728640.0,
            "flags": [],
            "fstype": "",
            "name": "",
            "num": 2,
            "size": 5242880.0,
            "unit": "kib"
        }
    ],
    "script": [
        "rm",
        "3"
    ]
}
suman1@ansible-server:~$ ansible suman -m command -a 'lsblk'
node2 | CHANGED | rc=0 >>
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    0   20G  0 disk 
├─sda1          8:1    0   10G  0 part 
└─sda2          8:2    0    5G  0 part 
sr0            11:0    1 10.3G  0 rom  /run/media/suman/RHEL-10-2-BaseOS-x86_64
nvme0n1       259:0    0   40G  0 disk 
├─nvme0n1p1   259:1    0  600M  0 part /boot/efi
├─nvme0n1p2   259:2    0    2G  0 part /boot
└─nvme0n1p3   259:3    0 37.4G  0 part 
  ├─rhel-root 253:0    0 33.5G  0 lvm  /
  └─rhel-swap 253:1    0  3.9G  0 lvm  [SWAP]
node1 | CHANGED | rc=0 >>
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    0   20G  0 disk 
├─sda1          8:1    0   10G  0 part 
└─sda2          8:2    0    5G  0 part 
sr0            11:0    1 10.3G  0 rom  /run/media/suman/RHEL-10-2-BaseOS-x86_64
nvme0n1       259:0    0   40G  0 disk 
├─nvme0n1p1   259:1    0  600M  0 part /boot/efi
├─nvme0n1p2   259:2    0    2G  0 part /boot
└─nvme0n1p3   259:3    0 37.4G  0 part 
  ├─rhel-root 253:0    0 33.5G  0 lvm  /
  └─rhel-swap 253:1    0  3.9G  0 lvm  [SWAP]
suman1@ansible-server:~$ 

------------------------------------------------------------------------------------
============================================
==============================================
-------------------------------------------------------

filesystem module

in Ansible, the ansible.builtin.filesystem module is used to create a filesystem on a disk partition or block device, such as XFS or EXT4.

create fstype=xfs in /dev/sda1

suman1@ansible-server:~$ ansible suman -m filesystem -a 'device=/dev/sda1 fstype=xfs'
[WARNING]: Collection community.general does not support Ansible version 2.16.16
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true
}
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true
}



---------------------------------
========================================
=======================================
-----------------------------------------
lvg module


in Ansible, the lvg module is used to create, modify, or remove LVM (Logical Volume Manager) volume groups on Linux servers.
For your RHEL 9/10 RHCE lab, you use it after preparing physical volumes with pvcreate or the Ansible lvol? Actually, the correct workflow is: prepare a physical volume, create a volume group using lvg, then create logical volumes using lvol.

---
suman1@ansible-server:~$ ansible suman -m command -a 'lsblk'
node2 | CHANGED | rc=0 >>
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    0   20G  0 disk 
├─sda1          8:1    0   10G  0 part 
└─sda2          8:2    0    5G  0 part 
sr0            11:0    1 10.3G  0 rom  /run/media/suman/RHEL-10-2-BaseOS-x86_64
nvme0n1       259:0    0   40G  0 disk 
├─nvme0n1p1   259:1    0  600M  0 part /boot/efi
├─nvme0n1p2   259:2    0    2G  0 part /boot
└─nvme0n1p3   259:3    0 37.4G  0 part 
  ├─rhel-root 253:0    0 33.5G  0 lvm  /
  └─rhel-swap 253:1    0  3.9G  0 lvm  [SWAP]
node1 | CHANGED | rc=0 >>
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    0   20G  0 disk 
├─sda1          8:1    0   10G  0 part 
└─sda2          8:2    0    5G  0 part 
sr0            11:0    1 10.3G  0 rom  /run/media/suman/RHEL-10-2-BaseOS-x86_64
nvme0n1       259:0    0   40G  0 disk 
├─nvme0n1p1   259:1    0  600M  0 part /boot/efi
├─nvme0n1p2   259:2    0    2G  0 part /boot
└─nvme0n1p3   259:3    0 37.4G  0 part 
  ├─rhel-root 253:0    0 33.5G  0 lvm  /
  └─rhel-swap 253:1    0  3.9G  0 lvm  [SWAP]
suman1@ansible-server:~$ ansible suman -m lvg -a 'pvs=/dev/sda2 vg=volgrp state=present'
[WARNING]: Collection community.general does not support Ansible version 2.16.16
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true
}
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true
}
suman1@ansible-server:~$ ansible suman -m command -a 'vgs'
node1 | CHANGED | rc=0 >>
  VG     #PV #LV #SN Attr   VSize  VFree 
  rhel     1   2   0 wz--n- 37.41g     0 
  volgrp   1   0   0 wz--n- <5.00g <5.00g
node2 | CHANGED | rc=0 >>
  VG     #PV #LV #SN Attr   VSize  VFree 
  rhel     1   2   0 wz--n- 37.41g     0 
  volgrp   1   0   0 wz--n- <5.00g <5.00g
suman1@ansible-server:~$ ansible suman -m lvg -a 'pvs=/dev/sda2,/dev/sda1 vg=volgrp state=present'
[WARNING]: Collection community.general does not support Ansible version 2.16.16
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true
}
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true
}
suman1@ansible-server:~$ ansible suman -m command -a 'vgs'
node2 | CHANGED | rc=0 >>
  VG     #PV #LV #SN Attr   VSize  VFree 
  rhel     1   2   0 wz--n- 37.41g     0 
  volgrp   2   0   0 wz--n- 14.99g 14.99g
node1 | CHANGED | rc=0 >>
  VG     #PV #LV #SN Attr   VSize  VFree 
  rhel     1   2   0 wz--n- 37.41g     0 
  volgrp   2   0   0 wz--n- 14.99g 14.99g
suman1@ansible-server:~$ ansible suman -m lvg -a 'vg=volgrp state=absent'
[WARNING]: Collection community.general does not support Ansible version 2.16.16
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true
}
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true
}
suman1@ansible-server:~$ ansible suman -m command -a 'vgs'
node2 | CHANGED | rc=0 >>
  VG   #PV #LV #SN Attr   VSize  VFree
  rhel   1   2   0 wz--n- 37.41g    0 
node1 | CHANGED | rc=0 >>
  VG   #PV #LV #SN Attr   VSize  VFree
  rhel   1   2   0 wz--n- 37.41g    0 
suman1@ansible-server:~$ 

----------------------------------------------------------------------------------------------------
===================
=================
======================
lvol module ----->

in Ansible, the community.general.lvol module is used to create, resize, and remove LVM (Logical Volume Manager) logical volumes on Linux servers such as RHEL 9 and RHEL 10.
You use it after creating a volume group with the lvg module.
1. What is the lvol module?
Suppose you have a 10 GB disk. You create a volume group named vgdata and then create a 2 GB logical volume named lvdata inside it.
Physical Volume (PV)
/dev/sdb1

Volume Group (VG)
vgdata — managed by lvg

Logical Volume (LV)
lvdata — 2 GB, managed by lvol


---------------------------


suman1@ansible-server:~$ ansible suman -m lvg -a 'pvs=/dev/sda2,/dev/sda1 vg=volgrp state=present'
[WARNING]: Collection community.general does not support Ansible version 2.16.16
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true
}
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true
}
suman1@ansible-server:~$ 
suman1@ansible-server:~$ 
suman1@ansible-server:~$ ansible suman -m lvol -a 'vg=volgrp lv=logvol size=4G state=present'
[WARNING]: Collection community.general does not support Ansible version 2.16.16
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "msg": ""
}
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "msg": ""
}
suman1@ansible-server:~$ ansible suman -m command -a 'lvs'
node2 | CHANGED | rc=0 >>
  LV     VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root   rhel   -wi-ao---- <33.50g                                                    
  swap   rhel   -wi-ao----   3.91g                                                    
  logvol volgrp -wi-a-----   4.00g                                                    
node1 | CHANGED | rc=0 >>
  LV     VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root   rhel   -wi-ao---- <33.50g                                                    
  swap   rhel   -wi-ao----   3.91g                                                    
  logvol volgrp -wi-a-----   4.00g                                                    
suman1@ansible-server:~$ ansible suman -m lvol -a 'vg=volgrp lv=logvol size=+2G state=present'
[WARNING]: Collection community.general does not support Ansible version 2.16.16
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "lv": "logvol",
    "size": 4.0,
    "vg": "volgrp"
}
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "lv": "logvol",
    "size": 4.0,
    "vg": "volgrp"
}
suman1@ansible-server:~$ ansible suman -m command -a 'lvs'
node2 | CHANGED | rc=0 >>
  LV     VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root   rhel   -wi-ao---- <33.50g                                                    
  swap   rhel   -wi-ao----   3.91g                                                    
  logvol volgrp -wi-a-----   6.00g                                                    
node1 | CHANGED | rc=0 >>
  LV     VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root   rhel   -wi-ao---- <33.50g                                                    
  swap   rhel   -wi-ao----   3.91g                                                    
  logvol volgrp -wi-a-----   6.00g                                                    
suman1@ansible-server:~$ ansible suman -m lvol -a 'vg=volgrp lv=test size=100%FREE state=present'
[WARNING]: Collection community.general does not support Ansible version 2.16.16
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "msg": ""
}
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "msg": ""
}
suman1@ansible-server:~$ ansible suman -m command -a 'lvs'
node2 | CHANGED | rc=0 >>
  LV     VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root   rhel   -wi-ao---- <33.50g                                                    
  swap   rhel   -wi-ao----   3.91g                                                    
  logvol volgrp -wi-a-----   6.00g                                                    
  test   volgrp -wi-a-----   8.99g                                                    
node1 | CHANGED | rc=0 >>
  LV     VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root   rhel   -wi-ao---- <33.50g                                                    
  swap   rhel   -wi-ao----   3.91g                                                    
  logvol volgrp -wi-a-----   6.00g                                                    
  test   volgrp -wi-a-----   8.99g                                                    
suman1@ansible-server:~$  ansible suman -m lvol -a 'vg=volgrp lv=test state=absent force=true'
[WARNING]: Collection community.general does not support Ansible version 2.16.16
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true
}
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true
}
suman1@ansible-server:~$ ansible suman -m command -a 'lvs'
node1 | CHANGED | rc=0 >>
  LV     VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root   rhel   -wi-ao---- <33.50g                                                    
  swap   rhel   -wi-ao----   3.91g                                                    
  logvol volgrp -wi-a-----   6.00g                                                    
node2 | CHANGED | rc=0 >>
  LV     VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root   rhel   -wi-ao---- <33.50g                                                    
  swap   rhel   -wi-ao----   3.91g                                                    
  logvol volgrp -wi-a-----   6.00g                                                    
suman1@ansible-server:~$ 

suman1@ansible-server:~$ ansible suman -m filesystem -a 'device=/dev/volgrp/logvol fstype=ext4'
[WARNING]: Collection community.general does not support Ansible version 2.16.16
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true
}
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true
}
suman1@ansible-server:~$ ansible suman -m command -a 'lsblk -f'
node1 | CHANGED | rc=0 >>
NAME              FSTYPE      FSVER            LABEL                   UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
sda                                                                                                                          
├─sda1            LVM2_member LVM2 001                                 kbZNid-BKxl-RcMP-6L4w-8Iu3-DH9H-d4CpDx                
│ └─volgrp-logvol ext4        1.0                                      71d4d38e-3247-41de-8504-a088e0b62656                  
└─sda2            LVM2_member LVM2 001                                 tR0qnR-ZE6P-heeg-Fz1K-W2NY-kUtj-XaB4yM                
  └─volgrp-logvol ext4        1.0                                      71d4d38e-3247-41de-8504-a088e0b62656                  
sr0               iso9660     Joliet Extension RHEL-10-2-BaseOS-x86_64 2026-05-11-20-11-35-00                       0   100% /run/media/suman/RHEL-10-2-BaseOS-x86_64
nvme0n1                                                                                                                      
├─nvme0n1p1       vfat        FAT32                                    33EC-8B77                               589.9M     1% /boot/efi
├─nvme0n1p2       xfs                                                  5ee0355f-fae0-44a0-bf77-5a5174e676be      1.5G    22% /boot
└─nvme0n1p3       LVM2_member LVM2 001                                 xux1AG-mWYL-dvkJ-N6Ks-adYf-8y3x-xgD0z1                
  ├─rhel-root     xfs                                                  4762de72-450d-43da-90e8-8660609c7dc9     27.3G    18% /
  └─rhel-swap     swap        1                                        c55023ad-0874-4d61-a582-29be01e94ddd                  [SWAP]
node2 | CHANGED | rc=0 >>
NAME              FSTYPE      FSVER            LABEL                   UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
sda                                                                                                                          
├─sda1            LVM2_member LVM2 001                                 ytOi7b-UUsK-RnKw-BIQ7-1MCI-dfdl-N3vGtI                
│ └─volgrp-logvol ext4        1.0                                      07909999-dea0-4d65-b942-3a303be14666                  
└─sda2            LVM2_member LVM2 001                                 k7EE4E-9hGq-UURI-VIZ1-p20a-5hQS-1CRmIK                
  └─volgrp-logvol ext4        1.0                                      07909999-dea0-4d65-b942-3a303be14666                  
sr0               iso9660     Joliet Extension RHEL-10-2-BaseOS-x86_64 2026-05-11-20-11-35-00                       0   100% /run/media/suman/RHEL-10-2-BaseOS-x86_64
nvme0n1                                                                                                                      
├─nvme0n1p1       vfat        FAT32                                    33EC-8B77                               589.9M     1% /boot/efi
├─nvme0n1p2       xfs                                                  5ee0355f-fae0-44a0-bf77-5a5174e676be      1.5G    22% /boot
└─nvme0n1p3       LVM2_member LVM2 001                                 xux1AG-mWYL-dvkJ-N6Ks-adYf-8y3x-xgD0z1                
  ├─rhel-root     xfs                                                  4762de72-450d-43da-90e8-8660609c7dc9     27.3G    18% /
  └─rhel-swap     swap        1                                        c55023ad-0874-4d61-a582-29be01e94ddd                  [SWAP]
suman1@ansible-server:~$ ansible suman -m command -a 'df -Th'
node1 | CHANGED | rc=0 >>
Filesystem            Type      Size  Used Avail Use% Mounted on
/dev/mapper/rhel-root xfs        34G  6.2G   28G  19% /
devtmpfs              devtmpfs  1.8G     0  1.8G   0% /dev
tmpfs                 tmpfs     1.8G   84K  1.8G   1% /dev/shm
efivarfs              efivarfs  256K   76K  176K  31% /sys/firmware/efi/efivars
tmpfs                 tmpfs     725M   20M  705M   3% /run
tmpfs                 tmpfs     1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
/dev/nvme0n1p2        xfs       2.0G  436M  1.6G  22% /boot
/dev/nvme0n1p1        vfat      599M  9.0M  590M   2% /boot/efi
tmpfs                 tmpfs     363M  192K  362M   1% /run/user/1000
/dev/sr0              iso9660    11G   11G     0 100% /run/media/suman/RHEL-10-2-BaseOS-x86_64
tmpfs                 tmpfs     363M   56K  363M   1% /run/user/0
tmpfs                 tmpfs     363M   56K  363M   1% /run/user/1001
node2 | CHANGED | rc=0 >>
Filesystem            Type      Size  Used Avail Use% Mounted on
/dev/mapper/rhel-root xfs        34G  6.2G   28G  19% /
devtmpfs              devtmpfs  1.8G     0  1.8G   0% /dev
tmpfs                 tmpfs     1.8G   84K  1.8G   1% /dev/shm
efivarfs              efivarfs  256K   76K  176K  31% /sys/firmware/efi/efivars
tmpfs                 tmpfs     725M   16M  709M   3% /run
tmpfs                 tmpfs     1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
/dev/nvme0n1p2        xfs       2.0G  436M  1.6G  22% /boot
/dev/nvme0n1p1        vfat      599M  9.0M  590M   2% /boot/efi
tmpfs                 tmpfs     363M  192K  362M   1% /run/user/1000
/dev/sr0              iso9660    11G   11G     0 100% /run/media/suman/RHEL-10-2-BaseOS-x86_64
tmpfs                 tmpfs     363M   56K  363M   1% /run/user/0
tmpfs                 tmpfs     363M   56K  363M   1% /run/user/1001
suman1@ansible-server:~$ ansible suman -m mount -a 'path=/opt src=/dev/volgrp/logvol fstype=ext4 opts=defaults state=mounted'
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup_file": "",
    "boot": "yes",
    "changed": true,
    "dump": "0",
    "fstab": "/etc/fstab",
    "fstype": "ext4",
    "name": "/opt",
    "opts": "defaults",
    "passno": "0",
    "src": "/dev/volgrp/logvol"
}
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup_file": "",
    "boot": "yes",
    "changed": true,
    "dump": "0",
    "fstab": "/etc/fstab",
    "fstype": "ext4",
    "name": "/opt",
    "opts": "defaults",
    "passno": "0",
    "src": "/dev/volgrp/logvol"
}
suman1@ansible-server:~$ ansible suman -m command -a 'df -Th'
node1 | CHANGED | rc=0 >>
Filesystem                Type      Size  Used Avail Use% Mounted on
/dev/mapper/rhel-root     xfs        34G  6.2G   28G  19% /
devtmpfs                  devtmpfs  1.8G     0  1.8G   0% /dev
tmpfs                     tmpfs     1.8G   84K  1.8G   1% /dev/shm
efivarfs                  efivarfs  256K   76K  176K  31% /sys/firmware/efi/efivars
tmpfs                     tmpfs     725M   20M  705M   3% /run
tmpfs                     tmpfs     1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
/dev/nvme0n1p2            xfs       2.0G  436M  1.6G  22% /boot
/dev/nvme0n1p1            vfat      599M  9.0M  590M   2% /boot/efi
tmpfs                     tmpfs     363M  192K  362M   1% /run/user/1000
/dev/sr0                  iso9660    11G   11G     0 100% /run/media/suman/RHEL-10-2-BaseOS-x86_64
tmpfs                     tmpfs     363M   56K  363M   1% /run/user/0
tmpfs                     tmpfs     363M   56K  363M   1% /run/user/1001
/dev/mapper/volgrp-logvol ext4      5.9G   24K  5.6G   1% /opt
node2 | CHANGED | rc=0 >>
Filesystem                Type      Size  Used Avail Use% Mounted on
/dev/mapper/rhel-root     xfs        34G  6.2G   28G  19% /
devtmpfs                  devtmpfs  1.8G     0  1.8G   0% /dev
tmpfs                     tmpfs     1.8G   84K  1.8G   1% /dev/shm
efivarfs                  efivarfs  256K   76K  176K  31% /sys/firmware/efi/efivars
tmpfs                     tmpfs     725M   20M  705M   3% /run
tmpfs                     tmpfs     1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
/dev/nvme0n1p2            xfs       2.0G  436M  1.6G  22% /boot
/dev/nvme0n1p1            vfat      599M  9.0M  590M   2% /boot/efi
tmpfs                     tmpfs     363M  192K  362M   1% /run/user/1000
/dev/sr0                  iso9660    11G   11G     0 100% /run/media/suman/RHEL-10-2-BaseOS-x86_64
tmpfs                     tmpfs     363M   56K  363M   1% /run/user/0
tmpfs                     tmpfs     363M   56K  363M   1% /run/user/1001
/dev/mapper/volgrp-logvol ext4      5.9G   24K  5.6G   1% /opt
suman1@ansible-server:~$ 

--------------------------------------------------------------------------------
================================================================================
====================================================================================
------------------------------------------------------------------------------------

cron module ---------------------->


n Ansible, the ansible.builtin.cron module is used to schedule tasks that run automatically at a specific time or interval on managed Linux servers.
It manages cron jobs in the user's crontab, just like editing the output of crontab -e manually.

-----------------

suman1@ansible-server:~$ ansible suman -m cron -a 'name=suman weekday=5 minute=45 hour=14 user=root job="echo suman bhai > /var/log/messages"'
node2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "envs": [],
    "jobs": [
        "suman"
    ]
}
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "envs": [],
    "jobs": [
        "suman"
    ]
}
```
