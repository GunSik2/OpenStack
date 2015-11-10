
ServerRaid Tool
- https://www-947.ibm.com/support/entry/portal/docdisplay?lndocid=MIGR-5073015

After creating raid group, check these command:
- cat /proc/scsi/scsi
- cat /proc/partitions
- fdisk -l

After checking a new drive, create disk partition and filesystem:
```
[root@control01 ~]# fdisk /dev/sdb
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0xa1d8b747.

Command (m for help): p

Disk /dev/sdb: 499.0 GB, 498999492608 bytes, 974608384 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0xa1d8b747

   Device Boot      Start         End      Blocks   Id  System

Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-974608383, default 2048):
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-974608383, default 974608383):
Using default value 974608383
Partition 1 of type Linux and of size 464.7 GiB is set

Command (m for help): p

Disk /dev/sdb: 499.0 GB, 498999492608 bytes, 974608384 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0xa1d8b747

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048   974608383   487303168   83  Linux

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
[root@control01 ~]#


[root@control01 ~]# mkfs.ext4 /dev/sdb1
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
30457856 inodes, 121825792 blocks
6091289 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=2271215616
3718 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
        102400000

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done

[root@control01 ~]#

```

