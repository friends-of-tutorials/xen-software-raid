# XenServer Software RAID installation (mdadm)

How to install a software raid on XenServer 7.6.2

## 1. Instruction

### 1.1 Installation XenServer

First install the XenServer as usual without the LVM repositories.

### 1.2 Check environment after XenServer installation

```bash
root$ lsblk 
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0 931,5G  0 disk 
├─sda1   8:1    0    18G  0 part /
├─sda2   8:2    0    18G  0 part 
├─sda3   8:3    0   512M  0 part 
├─sda5   8:5    0     4G  0 part /var/log
└─sda6   8:6    0     1G  0 part [SWAP]
sdb      8:32   0 931,5G  0 disk 
sdc      8:32   0 931,5G  0 disk 
sdd      8:48   0 931,5G  0 disk 
sr0     11:0    1  1024M  0 rom  
loop0    7:0    0    44M  1 loop /var/xen/xc-install
```

### 1.3 load kernel module

```bash
modprobe md_mod
modprobe raid1
```

### 1.4 Delete partition informations on /dev/sdb

```bash
sgdisk --zap-all /dev/sdb
sgdisk --mbrtogpt --clear /dev/sdb
```

### 1.5 Copy the partition table from /dev/sda to /dev/sdb

```bash
sgdisk -R /dev/sdb /dev/sda
```
 
### 1.6 Set partition types

```bash
sgdisk --typecode=1:fd00 /dev/sdb
sgdisk --typecode=2:fd00 /dev/sdb
sgdisk --typecode=3:ef02 /dev/sdb
sgdisk --typecode=5:fd00 /dev/sdb
sgdisk --typecode=6:fd00 /dev/sdb
```

### 1.6 Create RAID with /dev/sdb

```bash
yes|mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb1 missing
yes|mdadm --create /dev/md1 --level=1 --raid-devices=2 /dev/sdb2 missing
yes|mdadm --create /dev/md2 --level=1 --raid-devices=2 /dev/sdb3 missing
yes|mdadm --create /dev/md4 --level=1 --raid-devices=2 /dev/sdb5 missing
yes|mdadm --create /dev/md5 --level=1 --raid-devices=2 /dev/sdb6 missing
```

#### 1.6.1 Check RAID

 ```bash
root$ cat /proc/mdstat 
Personalities : [raid1] 
md5 : active raid1 sdb6[0]
      1047552 blocks super 1.2 [2/1] [U_]
      
md4 : active raid1 sdb5[0]
      4190208 blocks super 1.2 [2/1] [U_]
      
md2 : active raid1 sdb3[0]
      523712 blocks super 1.2 [2/1] [U_]
      
md1 : active raid1 sdb2[0]
      18857984 blocks super 1.2 [2/1] [U_]
      
md0 : active raid1 sdb1[0]
      18857984 blocks super 1.2 [2/1] [U_]
      
unused devices: <none>
```
 
### 1.7 Create the filesystem

```bash
mkfs.ext3 /dev/md0
mkfs.ext3 /dev/md4
mkswap /dev/md5
```

#### 1.7.1 Check the filesystem

```bash
root$ lsblk 
NAME    MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda       8:0    0 931,5G  0 disk  
├─sda1    8:1    0    18G  0 part  /
├─sda2    8:2    0    18G  0 part  
├─sda3    8:3    0   512M  0 part  
├─sda5    8:5    0     4G  0 part  /var/log
└─sda6    8:6    0     1G  0 part  [SWAP]
sdb       8:16   0 931,5G  0 disk  
├─sdb1    8:17   0    18G  0 part  
│ └─md0   9:0    0    18G  0 raid1 
├─sdb2    8:18   0    18G  0 part  
│ └─md1   9:1    0    18G  0 raid1 
├─sdb3    8:19   0   512M  0 part  
│ └─md2   9:2    0 511,4M  0 raid1 
├─sdb5    8:21   0     4G  0 part  
│ └─md4   9:4    0     4G  0 raid1 
└─sdb6    8:22   0     1G  0 part  
  └─md5   9:5    0  1023M  0 raid1 
sr0      11:0    1  1024M  0 rom   
loop0     7:0    0    44M  1 loop  /var/xen/xc-install
```

### 1.8 Mount the directories of /dev/sdb for preparation

```bash
mount /dev/md0 /mnt
mkdir -p /mnt/var/log
mount /dev/md4 /mnt/var/log
```

### 1.9 Clone data to raid devices from /dev/sda to /dev/sdb

```bash
cp -xa / /mnt
cp -xa /var/log /mnt/var
```

### 1.10 create mdadm config

```bash
echo "MAILADDR root" > /mnt/etc/mdadm.conf
echo "auto +imsm +1.x -all" >> /mnt/etc/mdadm.conf
echo "DEVICE /dev/sd*[a-z][1-9]" >> /mnt/etc/mdadm.conf
mdadm --detail --scan >> /mnt/etc/mdadm.conf
cp /mnt/etc/mdadm.conf /etc
```

#### 1.10.1 Check the mdadm config

```bash
root$ cat /etc/mdadm.conf
MAILADDR root
auto +imsm +1.x -all
DEVICE /dev/sd*[a-z][1-9]
ARRAY /dev/md0 metadata=1.2 name=xenserver-7:0 UUID=b6eafb52:53b1c0ed:03b029cd:ce0fa6bf
ARRAY /dev/md1 metadata=1.2 name=xenserver-7:1 UUID=7b4bae54:236b1e2f:7f1a7a0e:e4610009
ARRAY /dev/md2 metadata=1.2 name=xenserver-7:2 UUID=0c082894:f0f24c6b:43bf4045:0970c253
ARRAY /dev/md4 metadata=1.2 name=xenserver-7:4 UUID=4a064db1:0c8e8f42:76810417:5cf0ecbd
ARRAY /dev/md5 metadata=1.2 name=xenserver-7:5 UUID=30dfc30a:1c8031bf:93d8c760:24186100
```

#### 1.10.1 Check the fstab config

```bash
root$ cat /mnt/etc/fstab
LABEL=root-ocmtfp    /         ext3     defaults   1  1
LABEL=swap-ocmtfp          swap      swap   defaults   0  0
LABEL=logs-ocmtfp    /var/log         ext3     defaults   0  2
/opt/xensource/packages/iso/XenCenter.iso   /var/xen/xc-install   iso9660   loop,ro   0  0
```

### 1.11 Set new fstab with md devices

```bash
sed -i 's/LABEL=root-[a-zA-Z\-]*/\/dev\/md0/' /mnt/etc/fstab
sed -i 's/LABEL=swap-[a-zA-Z\-]*/\/dev\/md5/' /mnt/etc/fstab
sed -i 's/LABEL=logs-[a-zA-Z\-]*/\/dev\/md4/' /mnt/etc/fstab
cp /mnt/etc/fstab /etc
```

#### 1.11.1 Check the fstab config after adjustment

```bash
root$ cat /mnt/etc/fstab
/dev/md0    /         ext3     defaults   1  1
/dev/md5          swap      swap   defaults   0  0
/dev/md4    /var/log         ext3     defaults   0  2
/opt/xensource/packages/iso/XenCenter.iso   /var/xen/xc-install   iso9660   loop,ro   0  0
```
 
### 1.12 Set label from sda to md0

```bash
e2label /dev/sda1 |xargs -t e2label /dev/md0
```

### 1.13 Chroot the RAID to make some changes on it

```bash
mount --bind /dev /mnt/dev
mount --bind /sys /mnt/sys
mount --bind /proc /mnt/proc
#mount --bind /run /mnt/run
chroot /mnt /bin/bash
```

### 1.14 Backup the initrd img

```bash
cp /boot/initrd-$(uname -r).img /boot/initrd-$(uname -r).img.bak
```

### 1.15 Build new initrd img

```bash
dracut --mdadmconf --fstab --add="mdraid" --add-drivers="raid1" --force /boot/initrd-$(uname -r).img $(uname -r) -M
```

### 1.16 Set grub configs

```bash
sed -i 's/quiet/rd.auto rd.auto=1 rhgb quiet/' /boot/grub/grub.cfg
sed -i 's/LABEL=root-[a-zA-Z\-]*/\/dev\/md0/' /boot/grub/grub.cfg
sed -i '/search/ i\  insmod gzio part_msdos diskfilter mdraid1x' /boot/grub/grub.cfg
sed -i '/search/ c\  set root=(md/0)' /boot/grub/grub.cfg
```

### 1.17 Install grub loader on /dev/sdb

```bash
grub-install /dev/sdb
```

### 1.18 Close chroot

```bash
exit
```

### 1.19 Backup and copy the created initrd img and grub config

```bash
cp /boot/initrd-$(uname -r).img /boot/initrd-$(uname -r).img.bak
cp /mnt/boot/initrd-$(uname -r).img /boot/
cp /mnt/boot/grub/grub.cfg /boot/grub/grub.cfg
```

### 1.20 Reboot the system (make sure to start with /dev/sdb!)

```bash
reboot
```

### 1.21 Check if /dev/md2 is missing

```bash
root$ lsblk 
NAME    MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda       8:0    0 931,5G  0 disk  
├─sda1    8:1    0    18G  0 part  /
├─sda2    8:2    0    18G  0 part  
├─sda3    8:3    0   512M  0 part  
├─sda5    8:5    0     4G  0 part  /var/log
└─sda6    8:6    0     1G  0 part  [SWAP]
sdb       8:16   0 931,5G  0 disk  
├─sdb1    8:17   0    18G  0 part  
│ └─md0   9:0    0    18G  0 raid1 
├─sdb2    8:18   0    18G  0 part  
│ └─md1   9:1    0    18G  0 raid1 
├─sdb3    8:19   0   512M  0 part  # <- /dev/md2 is missing
├─sdb5    8:21   0     4G  0 part  
│ └─md4   9:4    0     4G  0 raid1 
└─sdb6    8:22   0     1G  0 part  
  └─md5   9:5    0  1023M  0 raid1 
sr0      11:0    1  1024M  0 rom   
loop0     7:0    0    44M  1 loop  /var/xen/xc-install
```

If so then set fd00, create a new raid and check the uuid in /etc/mdadm.conf

```bash
sgdisk --typecode=3:fd00 /dev/sdb
yes|mdadm --create /dev/md2 --level=1 --raid-devices=2 /dev/sdb3 missing
mdadm --detail --scan # vs.
cat /etc/mdadm.conf
vi /etc/mdadm.conf # change uuid if has changed
```

### 1.22 Delete partition informations on /dev/sda

```bash
sgdisk --zap-all /dev/sda
sgdisk --mbrtogpt --clear /dev/sda
```

### 1.23 Copy the partition table from sdb to sda

```bash
sgdisk -R /dev/sda /dev/sdb
```

### 1.23 Complete raid

```bash
mdadm -a /dev/md0 /dev/sda1
mdadm -a /dev/md1 /dev/sda2
mdadm -a /dev/md2 /dev/sda3
mdadm -a /dev/md4 /dev/sda5
mdadm -a /dev/md5 /dev/sda6
```

### 1.24 Check syncing RAID

```bash
root$ cat /proc/mdstat
Personalities : [raid1] 
md1 : active raid1 sda2[2] sdb2[0]
      18857984 blocks super 1.2 [2/2] [UU]
      
md4 : active raid1 sdb5[0] sda5[2]
      933114560 blocks super 1.2 [2/2] [UU]
      [=>...................]  resync =  7.2% (67247616/933114560) finish=136.2min speed=105941K/sec
      bitmap: 7/7 pages [28KB], 65536KB chunk
      
md5 : active raid1 sda6[2] sdb6[0]
      1047552 blocks super 1.2 [2/2] [UU]
      
md0 : active raid1 sdb1[0] sda1[2]
      18857984 blocks super 1.2 [2/2] [UU]
      
md2 : active raid1 sdb3[0] sda3[2]
      523712 blocks super 1.2 [2/2] [UU]
      
unused devices: <none>
```

### 1.25 Install grub on /dev/sda

```bash
grub-install /dev/sda
```

### 1.26 Create LVM partitions on /dev/sda and /dev/sdb

```bash
gdisk /dev/sda
n -> 4 -> ENTER -> ENTER -> FD00 -> w
gdisk /dev/sdb
n -> 4 -> ENTER -> ENTER -> FD00 -> w
```

### 1.27 Create LVM RAID I

```bash
yes|mdadm --create /dev/md3 --level=1 --raid-devices=2 /dev/sda4 /dev/sdb4
```

### 1.28 Maybe create LVM partitions on /dev/sdc and /dev/sdd

```bash
gdisk /dev/sdc
n -> 4 -> ENTER -> ENTER -> FD00 -> w
gdisk /dev/sdd
n -> 4 -> ENTER -> ENTER -> FD00 -> w
```
 
### 1.29 Maybe create LVM RAID II

```bash
yes|mdadm --create /dev/md6 --level=1 --raid-devices=2 /dev/sdc1 /dev/sdd1
```

### 1.30 Check partitions

```bash
root$ lsblk 
NAME    MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda       8:0    0 931,5G  0 disk  
├─sda1    8:1    0    18G  0 part  
│ └─md0   9:0    0    18G  0 raid1 /
├─sda2    8:2    0    18G  0 part  
│ └─md1   9:1    0    18G  0 raid1 
├─sda3    8:3    0   512M  0 part  
│ └─md2   9:2    0 511,4M  0 raid1 
├─sda4    8:4    0   890G  0 part  
│ └─md3   9:3    0 889,9G  0 raid1 
├─sda5    8:5    0     4G  0 part  
│ └─md4   9:4    0     4G  0 raid1 /var/log
└─sda6    8:6    0     1G  0 part  
  └─md5   9:5    0  1023M  0 raid1 [SWAP]
sdb       8:16   0 931,5G  0 disk  
├─sdb1    8:17   0    18G  0 part  
│ └─md0   9:0    0    18G  0 raid1 /
├─sdb2    8:18   0    18G  0 part  
│ └─md1   9:1    0    18G  0 raid1 
├─sdb3    8:19   0   512M  0 part  
│ └─md2   9:2    0 511,4M  0 raid1 
├─sdb4    8:20   0   890G  0 part  
│ └─md3   9:3    0 889,9G  0 raid1 
├─sdb5    8:21   0     4G  0 part  
│ └─md4   9:4    0     4G  0 raid1 /var/log
└─sdb6    8:22   0     1G  0 part  
  └─md5   9:5    0  1023M  0 raid1 [SWAP]
sdc       8:32   0 931,5G  0 disk  
└─sdc1    8:33   0 931,5G  0 part  
  └─md6   9:6    0 931,4G  0 raid1 
sdd       8:48   0 931,5G  0 disk  
└─sdd1    8:49   0 931,5G  0 part  
  └─md6   9:6    0 931,4G  0 raid1 
sr0      11:0    1  1024M  0 rom   
loop0     7:0    0    44M  1 loop  /var/xen/xc-install
```

### 1.31 Check syncing RAID

```bash
root$ cat /proc/mdstat
Personalities : [raid1] 
md6 : active raid1 sdd1[1] sdc1[0]
      976630464 blocks super 1.2 [2/2] [UU]
      [>....................]  resync =  1.3% (13116544/976630464) finish=146.0min speed=109984K/sec
      bitmap: 8/8 pages [32KB], 65536KB chunk

md3 : active raid1 sdb4[1] sda4[0]
      933114560 blocks super 1.2 [2/2] [UU]
      [=>...................]  resync =  7.2% (67247616/933114560) finish=136.2min speed=105941K/sec
      bitmap: 7/7 pages [28KB], 65536KB chunk

md1 : active raid1 sda2[2] sdb2[0]
      18857984 blocks super 1.2 [2/2] [UU]
      
md4 : active raid1 sdb5[0] sda5[2]
      4190208 blocks super 1.2 [2/2] [UU]
      
md5 : active raid1 sda6[2] sdb6[0]
      1047552 blocks super 1.2 [2/2] [UU]
      
md0 : active raid1 sdb1[0] sda1[2]
      18857984 blocks super 1.2 [2/2] [UU]
      
md2 : active raid1 sdb3[0] sda3[2]
      523712 blocks super 1.2 [2/2] [UU]
      
unused devices: <none>
```

### 1.32 Create LVM I

```bash
pvcreate /dev/md3
xe sr-create type=lvm content-type=user device-config:device=/dev/md3 name-label="Local Storage 1"
```

### 1.33 Maybe create lvm II

```bash
pvcreate /dev/md6
xe sr-create type=lvm content-type=user device-config:device=/dev/md6 name-label="Local Storage 2"
```

## A. Authors

* Björn Hempel <bjoern@hempel.li> - _Initial work_ - [https://github.com/bjoern-hempel](https://github.com/bjoern-hempel)

## B. Licence

This tutorial is licensed under the MIT License - see the [LICENSE.md](/LICENSE.md) file for details

## C. Closing words

Have fun! :)
