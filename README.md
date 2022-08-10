# IRIX-disk-partitioning
IRIX disk partitioning

Main references:<br>
https://techpubs.jurassic.nl/manuals/0650/admin/IA_DiskFiles/sgi_html/index.html
http://ibgwww.colorado.edu/~lessem/psyc5112/usail/peripherals/

Read chapter 1 on the first link to understand basic concepts:<br>
https://techpubs.jurassic.nl/manuals/0650/admin/IA_DiskFiles/sgi_html/ch01.html

Let's begin with an empty disk with SCSI address 1, what contains its volume header?:
```
prtvtoc /dev/dsk/dks0d1vol
Segmentation fault (core dumped)
```

The disk doesn't have volume header, can we edit it?
```
dvhtool /dev/rdsk/dks0d1vh
dvhtool: can't open volume header /dev/rdsk/dks0d1vh
Volume? (/dev/rdsk/dks0d1vh)
```
Note: disk devices are on /dev/rdsk, list then running:
```
ls -l /dev/rdsk/*
```

In fact, the disk is empty. Let's partition it:
```
fx -x
fx version 6.5, Oct  6, 2003
fx: "device-name" = (dksc)
fx: ctlr# = (0)
fx: drive# = (1)
fx: lun# = (0)
...opening dksc(0,1,0)
...drive selftest...OK
Scsi drive type == HP      9.10GB A 68-S94CS94C
warning: partition 8 not of type volhdr, volhdr is too small
NOTE:  existing partitions are inconsistent with drive geometry

----- please choose one (? for help, .. to quit this menu)-----
[exi]t             [d]ebug/           [l]abel/           [a]uto
[b]adblock/        [exe]rcise/        [r]epartition/
```

It doesn't contain partitions:
```
fx> r

----- partitions-----
part  type        blocks            Megabytes   (base+size)
 10: volume        0 + 17773524       0 + 8678
warning: partition 8 not of type volhdr, volhdr is too small

capacity is 17773524 blocks

----- please choose one (? for help, .. to quit this menu)-----
[ro]otdrive           [o]ptiondrive         [e]xpert
[u]srrootdrive        [re]size
```

#### Partition templates
1. [ro]otdrive. This schema contains:
- partition 0 (root), boot partition containing all files
- partition 1 for swap
- partition 8 for volume header. It's an "odd" partition containing several files and the partition schema, device parameters and some programs (check https://techpubs.jurassic.nl/manuals/0650/admin/IA_DiskFiles/sgi_html/ch01.html#LE18926-PARENT)
- partition 10 that represents the whole disk
```
fx/repartition> ro

fx/repartition/rootdrive: type of data partition = (xfs)
Warning: you will need to re-install all software and restore user data
from backups after changing the partition layout.  Changing partitions
will cause all data on the drive to be lost.  Be sure you have the drive
backed up if it contains any user data.  Continue? yes

----- partitions-----
part  type        blocks            Megabytes   (base+size)
  0: xfs      266240 + 17507284     130 + 8548
  1: raw        4096 + 262144         2 + 128 
  8: volhdr        0 + 4096           0 + 2   
 10: volume        0 + 17773524       0 + 8678

capacity is 17773524 blocks

----- please choose one (? for help, .. to quit this menu)-----
[ro]otdrive           [o]ptiondrive         [e]xpert
[u]srrootdrive        [re]size
```

2. [u]srrootdrive. This schema contains the same partitions than [ro]otdrive and:
- partition 6 (usr), to separate filesystems root and usr
```
fx/repartition> u

fx/repartition/usrrootdrive: type of data partition = (xfs)

----- partitions-----
part  type        blocks            Megabytes   (base+size)
  0: xfs      266240 + 102400       130 + 50   
  1: raw        4096 + 262144         2 + 128 
  6: xfs      368640 + 17404884     180 + 8498
  8: volhdr        0 + 4096           0 + 2   
 10: volume        0 + 17773524       0 + 8678

capacity is 17773524 blocks

----- please choose one (? for help, .. to quit this menu)-----
[ro]otdrive           [o]ptiondrive         [e]xpert
[u]srrootdrive        [re]size
fx/repartition>
```

3. [o]ptiondrive.
- partition 7 for general data store, useful for example when we have a secondary disk for data
```
fx/repartition> o 

fx/repartition/optiondrive: type of data partition = (xfs)

----- partitions-----
part  type        blocks            Megabytes   (base+size)
  7: xfs        4096 + 17769428       2 + 8676
  8: volhdr        0 + 4096           0 + 2   
 10: volume        0 + 17773524       0 + 8678

capacity is 17773524 blocks

----- please choose one (? for help, .. to quit this menu)-----
[ro]otdrive           [o]ptiondrive         [e]xpert
[u]srrootdrive        [re]size
fx/repartition>
```

Now let's make the partitioning more complicated. What about a root partition and another one for data? For example with big disks using EFS, where the maximum partition is 8 GB, we could have:
- one root EFS partition of 8 GB and another one of type 7 EFS of 8 GB
- one root EFS partition of 2 GB and another one of type 7 XFS containing the remaining disk space

As practical example, let's partition a disk using two XFS partitions:

1. We start from a [ro]otdrive configuration:
```
fx/repartition> ro

fx/repartition/rootdrive: type of data partition = (xfs)

----- partitions-----
part  type        blocks            Megabytes   (base+size)
  0: xfs      266240 + 17507284     130 + 8548
  1: raw        4096 + 262144         2 + 128 
  8: volhdr        0 + 4096           0 + 2   
 10: volume        0 + 17773524       0 + 8678

capacity is 17773524 blocks

----- please choose one (? for help, .. to quit this menu)-----
[ro]otdrive           [o]ptiondrive         [e]xpert
[u]srrootdrive        [re]size
fx/repartition>
```

2. Let's watch the column "Megabytes (base+size)":
- partition 8 takes from MB 0 to MB 1 (2 MB in total)
- partition 1 takes from MB 2 to MB 129 (128 MB in total)
- partition 0 takes from MB 130 to MB 8677 (8548 MB in total)

3. Let's make partition 0 smaller, 2 GB, and when it asks for partition 7, we'll start from where partition 0 ends and use the remaining space. It's a manual method but it works:
- partition 7: we start at 2178 which is where root ends (130 + 2048) and specify the maximum size possible:
```
----- please choose one (? for help, .. to quit this menu)-----
[ro]otdrive           [o]ptiondrive         [e]xpert
[u]srrootdrive        [re]size
fx/repartition> e

Enter .. when done
fx/repartition/expert: change partition = (0)
before:  type xfs        block  266240,       130 MB
                         len:  17507284 blks, 8548 MB
fx/repartition/expert: partition type = (xfs)
fx/repartition/expert: base in megabytes = (130)
fx/repartition/expert: size in megabytes (max 8548) = (8548) 2048
 after:  type xfs        block  266240,       130 MB
                         len:  4194304 blks, 2048 MB
fx/repartition/expert: change partition = (1)
before:  type raw        block    4096,         2 MB
                         len:   262144 blks,  128 MB
fx/repartition/expert: partition type = (raw)
fx/repartition/expert: base in megabytes = (2)
fx/repartition/expert: size in megabytes (max 8676) = (128)
fx/repartition/expert: change partition = (6)
before:  type xfs        block       0,         0 MB
                         len:        0 blks,    0 MB
fx/repartition/expert: partition type = (xfs)
fx/repartition/expert: base in megabytes = (0)
fx/repartition/expert: size in megabytes (max 8678) = (0)
 after:  type xfs        block       0,         0 MB
                         len:        0 blks,    0 MB
fx/repartition/expert: change partition = (7)
before:  type xfs        block       0,         0 MB
                         len:        0 blks,    0 MB
fx/repartition/expert: partition type = (xfs)
fx/repartition/expert: base in megabytes = (0) 2178
fx/repartition/expert: size in megabytes (max 6500) = (0) 6500
 after:  type xfs        block 4460544,      2178 MB
                         len:  13312980 blks, 6500 MB
fx/repartition/expert: change partition = (8)
before:  type volhdr     block       0,         0 MB
                         len:     4096 blks,    2 MB
fx/repartition/expert: partition type = (volhdr)
fx/repartition/expert: base in megabytes = (0)
fx/repartition/expert: size in megabytes (max 8678) = (2)

----- partitions-----
part  type        blocks            Megabytes   (base+size)
  0: xfs      266240 + 4194304      130 + 2048
  1: raw        4096 + 262144         2 + 128 
  7: xfs     4460544 + 13312980    2178 + 6500
  8: volhdr        0 + 4096           0 + 2   
 10: volume        0 + 17773524       0 + 8678

capacity is 17773524 blocks

----- please choose one (? for help, .. to quit this menu)-----
[ro]otdrive           [o]ptiondrive         [e]xpert
[u]srrootdrive        [re]size
fx/repartition>
```

4. Let's check everything is correct:
```
prtvtoc /dev/dsk/dks0d1vol
* /dev/dsk/dks0d1vol (bootfile "/unix")
*     512 bytes/sector
Partition  Type  Fs   Start: sec    Size: sec   Mount Directory
 0          xfs           266240      4194304 
 1          raw             4096       262144 
 7          xfs          4460544     13312980 
 8       volhdr                0         4096 
10       volume                0     17773524 
```

5. Let's format partitions:
- note: dks0d1s7 corresponds to controller 0, SCSI address 1 and partition 7
```
mkfs_xfs /dev/dsk/dks0d1s7
```

for older IRIX versions:
```
mkfs_efs /dev/dsk/dks0d1s7
```

Note: in /dev/dsk are the disk devices, you can see them running:
```
ls -l /dev/dsk/*
```

6. To temporarily mount the partition we can use mnt:
```
mount /dev/dsk/dks0d1s7 /mnt
```

7. To mount the partition permanently we can do it from the desktop with the Filesystem Manager or by adding in fstab:
```
/dev/dsk/dks0d1s7   /punto_montaje     xfs   rw  0 0
```

or if using EFS:
```
/dev/dsk/dks0d1s7   /punto_montaje     efs   rw  0 0
```
