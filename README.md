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
