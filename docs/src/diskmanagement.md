# HardDisk

------------------------------------------------------------------------

## Master Boot Record

```shell
# backup mbr
# startup --> BIOS --> MBR(512 byte sector of the device) --> BootManager(f.e grub,lilo)
fdisk -l
dd if=/dev/sdX of=mbr.dump bs=512 count=1
od -t x1 mbr.dump | tail -6
# restore mbr
# dd if=mbr.dump of=/dev/sdX bs=512 count=1
```

## Quota

```shell
yum install -y quota
vim /etc/fstab
# /dev/mapper/vg_centos-lv_home /home defaults,usrquota,grpquota
mount -o remount /home
# turn it on
quotacheck -cugv /home
# add new user and set quota for it
useradd -s /bin/bash -m -d /home/dojomi -c "dojomi" -g root dojomi
setquota -u dojomi 0 MB 0 0 /dev/mapper/vg_centos-lv_home
# 0 soft quota block size
# MB hard quota
# 0 soft quota file amount
# 0 hard quota file amount
# see report
repquota /home
# make some changes manually
edquota dojomi
```

## Rescue Volume

```shell
# get UUID for the correct disk g.e /dev/sda1
ls -l /dev/disk/by-uuid || lsblk

dd if=/.iso of=/usb bs=8M status=progress && sync

mount /dev/sda1 /mnt
mount -o bind /proc /mnt
mount -o bind /sys /mnt
mount -o bind /dev /mnt

chroot /mnt
# change /etc/fstab
```

## Logical Volume Manager

```shell
# added new virtual partitions
cat /proc/partitions
```

```shell
# fdisk
fdisk /dev/sdX
# m
# d
# n
# p
# t
# 8e
# w
mkfs.ext4 /dev/sdX1
```

```shell
# disks >2T
# list all partitions
parted -l
parted 
select /dev/sdX
# delete old partitions
# rm <number>
mklabel gpt
# make some partitions
mkpart primary ext4 0% 100%
set 1 lvm on 
print
quit
# make sure that the kernel aware partitions
partprobe /dev/sdX
```

```shell
# copy directly to all of other disks
sfdisk /dev/sdX | sfdisk /dev/sdX
```

```shell
# create physical volume [harddrive]
pvcreate -v /dev/sdX1
pvs || pvdisplay
# create new volume group and add harddrives
vgcreate vg_all /dev/sdX1
vgs || vgdisplay
# extend volume group with a new disk
vgextend vg_all /dev/sdX1
pvs
vgs vg_all
# change groupname
vgrename vg_all vgroup
vgs   
```

```shell
# create logical volume
lvcreate -n lv_data -L50M vg_all
mkfs.ext4 /dev/vg_all/lv_data
lvs || lvdisplay
# show the current status
lvscan
# change name
lvrename vg_all lv_data lv_files || lvrename /dev/vg_all/lv_data /dev/vg_all/lv_files
# use it
mkdir -p /media/vg_all/lv_data
ln -s /media/vg_all HOME/Desktop
mount -t ext4 /dev/vg_all/lv_data /media/vg_all/lv_data
```

```shell
# reseize logical volume
lvextend -L+50M /dev/vg_all/lv_data
e2fsck -f /dev/vg_all/lv_data
resize2fs /dev/vg_all/lv_data
```

```shell
# reduce logical volume
lvreduce -L50M /dev/vg_all/lv_data
e2fsck -f /dev/vg_all/lv_data
resize2fs /dev/vg_all/lv_data
# remove a disk
pvs
pvmove /dev/sdX1
lvs
# remove volume from group
vgremove vg_all /dev/sdX1
pvs
# remove physical volume
pvremove /dev/sdX1
pvs
shutdown -h now
# disk can be removed   
```

```shell
# remove created logical volume
lvremove vg_all /dev/vg_all/lv_data
# remove group
vgremove vg_all
pvs
```

```shell
# swap
lvcreate -n lv_swap -L50M vg_all
mkswap /dev/vg_all/lv_swap
swapon /dev/vg_all/lv_swap
swapon -s
```

```shell
# snapshot
# create some data on lv_data
mount -t ext4 /dev/vg_all/lv_data /media/vg_all/lv_data
cd /media/vg_all/lv_data
dd if=/dev/urandom of=data.iso bs=512 count=1
mkfs.ext4 /dev/vg_all/lv_lock
mkdir /media/vg_all/lv_backup
lvcreate --snapshot --name lv_backup -L100M /dev/vg_all/lv_data
lvscan
mount -t ext4 /dev/vg_all/lv_backup /media/vg_all/lv_backup
# compress it with same permisssions
tar -pzcf lv_backup-tar.gz /media/vg_all/lv_backup/*
# remove it
# umount /media/vg_all/*
# lvremove /dev/vg_all/lv_backup
```

## Software-Raid-Controller

```shell
yum install -y mdadm
# fdisk
fdisk -l
fsidk /dev/sda
# n
# 1
# t
# fd # hex for raid auto
# w

# parted 
parted /dev/sdb
mklabel gpt
mkpart primary 0% 100%
se1 raid on
print
```

```shell
# make sure that the kernel aware partitions
partprobe /dev/sdX1 
```

```shell
mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sdX1 /dev/sdX1 /dev/sdX1 
mkfs.ext4 /dev/md0
mkdir /media/raid5 && ln -s /media/raid5 /home/dojomi/Desktop
mount -t ext4 /dev/md0 /media/raid5
cat /proc/mdstat || mdadm -D /dev/md0
# mount automatically 
cat >> /etc/fstab <<eof
/dev/md0    /media/raid5  ext4    defaults    0   0
eof
```

```shell
# simulate a manual fail & rm failed disk
mdadm /dev/md0 -f /dev/sdX1
# faulty spare on /dev/sdX1
mdadm -D /dev/md0
# hot remove failed disk
mdadm /dev/md0 -r /dev/sdX1
mdadm -D /dev/md0
```

```shell
# extend raid with a fresh sata
mdadm -D /dev/md0 | grep -e "Size"
mdadm /dev/md0 -a /dev/sdX1
watch cat /proc/mdstat
# reshape modus (grow + resize)
mdadm --grow -l5 /dev/md0
resize2fs /dev/md0
mdadm -D /dev/md0 | grep -e "Size"
```

```shell
# status alert messages via mail
mdadm --detail --scan --verbose > /etc/mdadm.conf
echo 'MAILADDR raid-admin@domain' >> /etc/mdadm.conf
```

```shell
# stop & remove the array
# unmount device /media/raid5 if exist
mdadm -S /dev/md0
cat /proc/mdstat
```

## LSI Hardware-RAID-Controller

```shell
# find out if there is a hw-raid
lspci -vv | grep -i raid

# check out the disk
megacli -PDList -aAll | egrep "Enclosure Device ID:|Slot Number:|Inquiry Data:"
# Enclosure Device ID: 252
# Slot Number: 4
# Enclosure Device ID: 252
# Slot Number: 5

# delete all hw-configurations
megacli -CfgLdDel -LALL -aALL
megacli -PDList -aN

megacli -CfgLdAdd -r1 [252:4,252:5] [WB] [RA] [Cached] -aN
# [WB] = write cache activate
# [RA] = read ahead activate
# [Cached] = Read cached activate

# write cache deactivieren
megacli -LDSetProp CachedBadBBU -LALL -aALL

# show information
megacli -LDInfo -LAll -aAll

# activate automatic rebuild
megacli -AdpAutoRbld -Dsply -aN
megacli -AdpAutoRbld -Enbl -aN

# save the configuration of the controller config
megacli -CfgSave -f raidcfg.txt -aN

# restore old config
megacli -CfgRestore -f raidcfg.txt -aN
```

**disk failure handling**

```shell
# -aN = adapter parameter f.e -a0
# E:S = enclosure Device ID:slot Number ( [252:2] -a0 )

# check if rebuild is enabled
megacli -AdpAutoRbld -Dsply -aN 

# save the default raid configuration
megacli -CfgSave -f raidcfg.txt -aN

# get E:S of the disk and device id for smartctl and check the SMART alert to find out which disk is not properly working
megacli -PDList -aAll | egrep "Enclosure Device ID:|Slot Number:|Inquiry Data:|Device Id:|Drive has flagged a S.M.A.R.T alert :"

# get all relevant information
smartctl -d sat+megaraid,DEVICE_ID -a /dev/sdX

# mark disk as offline if it's not done automatically and as missing. Afterwards remove it from raid
megacli -pdoffline -physdrv[E:S] -aAll
megacli -pdmarkmissing -physdrv[E:S] -aAll   
megacli -pdprprmv -physdrv[E:S] -aN          

# show rebuild process
megacli -PDRbld -ShowProg -PhysDrv[E:S] -aN

# scan for the new disk if it will not showed 
megacli -CfgForeign -Scan -aN
```
