# Storage

------------------------------------------------------------------------

## Glusterfs

Storage Replication with 2 Bricks (centvm1 + centvm2)

```shell
# centvm1
echo -e "192.168.1.11 centvm1 centvm1.labs.local" >> /etc/hosts
echo -e "192.168.1.12 centvm2 centvm2.labs.local" >> /etc/hosts
nmcli general hostname centvm1.labs.local

# centvm2
echo -e "192.168.1.12 centvm2 centvm2.labs.local" >> /etc/hosts
echo -e "192.168.1.11 centvm1 centvm1.labs.local" >> /etc/hosts
nmcli general hostname centvm1.labs.local

# centvm1+centvm2
yum install -y epel-release
curl http://download.gluster.org/pub/gluster/glusterfs/LATEST/EPEL.repo/glusterfs-epel.repo -o /etc/yum.repos.d/glusterfs-epel.repo 
yum repolist
yum --enablerepo=glusterfs-epel -y install glusterfs-server
systemctl start glusterd
systemctl enable glusterd
# open some specific ports 
# cluster daemon
firewall-cmd --permanent --add-port=24007/tcp
# cluster management
firewall-cmd --permanent --add-port=24008/tcp
# each brick for every volume requires it's own port/range 49152-49153
firewall-cmd --permanent --add-port=24009/tcp
firewall-cmd --permanent --add-port=24010/tcp
# gluster nfs
firewall-cmd --permanent --add-port=38465-38467/tcp
# portmapper
firewall-cmd --permanent --add-port={111/udp,111/tcp}
firewall-cmd --reload
mkdir -p /data/brick/glustervol
# centvm1
gluster peer probe centvm2
gluster peer status
# centvm2
gluster peer probe centvm1
gluster peer status
# centvm1+centvm2
# make a mountpoint
mkdir /media/glustervol
```

Brick1 (centvm1)

```shell
#under sudo you have to use command force at the end
gluster volume create glustervol replica 2 transport tcp centvm1:/data/brick/glustervol centvm2:/data/brick/glustervol
gluster vol start glustervol
gluster vol info
# Volume Name: glustervol
# Type: Replicate
# Volume ID: 01fc1ed2-25a5-4ead-b882-0c4182cdae3d
# Status: Started
# Number of Bricks: 1 x 2 = 2
# Transport-type: tcp
# Bricks:
# Brick1: centvm1:/data/brick/glustervol
# Brick2: centvm2:/data/brick/glustervol
# Options Reconfigured:
# transport.address-family: inet
# performance.readdir-ahead: on
mount -t glusterfs centvm1:/glustervol /media/glustervol
mount
# automount volume 
echo -e "centvm1:/glustervol /media/glustervol glusterfs defaults,_netdev 0 0" >> /etc/fstab
touch /media/glustervol/new
```

Brick2 (centvm2)

```shell
mount -t glusterfs centvm2:/glustervol /media/glustervol
ls -la /media/glustervol
# iptables -F then it sync on my side
# automount volume 
echo -e "centvm1:/glustervol /media/glustervol glusterfs defaults,_netdev 0 0" >> /etc/fstab
gluster vol status
```

Client (clientvm)

```shell
echo -e "192.168.1.11 centvm1 centvm1.labs.local" >> /etc/hosts
echo -e "192.168.1.12 centvm2 centvm2.labs.local" >> /etc/hosts
nmcli general hostname clientvm.labs.local

curl http://download.gluster.org/pub/gluster/glusterfs/LATEST/EPEL.repo/glusterfs-epel.repo -o /etc/yum.repos.d/glusterfs-epel.repo 
yum repolist
yum --enablerepo=glusterfs-epel -y install glusterfs-fuse
mount -t glusterfs centvm1:/glustervol /media/glustervol
mount
# automount volume 
echo -e "centvm1:/glustervol /media/glustervol glusterfs defaults,_netdev 0 0" >> /etc/fstab
```

Delete the created volume

```shell
gluster vol stop glustervol
gluster vol delete glustervol
```

## iscsi

server-side

```shell
yum install -y lvm2
lsblk
# NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
# sda      8:0    0   10G  0 disk 
# ├─sda1   8:1    0  500M  0 part /boot
# ├─sda2   8:2    0    1G  0 part [SWAP]
# └─sda3   8:3    0  8.5G  0 part /
# sdb      8:16   0    8G  0 disk 
# sr0     11:0    1 1024M  0 rom
fdisk /dev/sdb
# Device Boot      Start         End      Blocks   Id  System
# /dev/sdb1         2099200    16777215     7339008   8e  Linux --> 100%FREE    
partprobe
pvcreate /dev/sdb1
vgcreate vg_iscsi /dev/sdb1
lvcreate -l 100%FREE -n lv_srv vg_iscsi
lvs
yum -y install targetcli
systemctl enable target.service
targetcli
cd backstores/block
create disk1 /dev/mapper/vg_iscsi-lv_srv
# create an iqn
cd / 
cd iscsi
create iqn.2016-04.virt.centos:srv
# create acl
cd iqn.2016-04.virt.centos:srv/tpg1/acls
create iqn.2016-04.virt.centos:srv
# create luns
cd .. 
cd luns
create /backstores/block/disk1
cd / 
exit
cat /etc/target/saveconfig.json
systemctl start target.service
netstat -nltp
firewall-cmd --permanent --add-port=3260/tcp
firewall-cmd --reload
```

client-side

```shell
yum install -y iscsi-initiator-utils lvm2
vim /etc/iscsi/initiatorname.iscsi
# InitiatorName=iqn.2016-04.virt.centos:srv
systemctl enable iscsid
man iscsiadm
iscsiadm -m discovery -t st -p 192.168.1.11:3260
# 192.168.1.11:3260,1 iqn.2016-04.virt.centos:srv
iscsiadm -m node -T iqn.2016-04.virt.centos:srv -p 192.168.1.11:3260 -l
lsblk || dmesg
# [ 3430.128127] sd 3:0:0:0: [sdb] Attached SCSI disk
pvcreate /dev/sdb
vgcreate vg_iscsi /dev/sdb
lvcreate -l 100%FREE -n lv_client vg_iscsi
lvs
mkfs.xfs /dev/mapper/vg_iscsi-lv_client 
mkdir /media/data_iscsi
mount -t xfs /dev/mapper/vg_iscsi-lv_client /media/data_iscsi
mount || df -h
# /dev/mapper/vg_iscsi-lv_client on /media/data_iscsi type xfs (rw,relatime,seclabel,attr2,inode64,noquota)
# automount
vim /etc/fstab
/dev/mapper/vg_iscsi-lv_client  /media/data_iscsi   xfs _netdev 1   2
# or use blkid for UUID
```

server-side

```shell
# reboot server once to get lvm entry from client-side
reboot
lsblk
# NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
# sda                        8:0    0   10G  0 disk 
# ├─sda1                     8:1    0  500M  0 part /boot
# ├─sda2                     8:2    0    1G  0 part [SWAP]
# └─sda3                     8:3    0  8.5G  0 part /
# sdb                        8:16   0    8G  0 disk 
# └─sdb1                     8:17   0    8G  0 part 
#   └─vg_iscsi-lv_srv      253:0    0    8G  0 lvm  
#     └─vg_iscsi-lv_client 253:1    0    8G  0 lvm  
# sr0                       11:0    1 1024M  0 rom  
mkdir /media/data_iscsi
mount -t xfs /dev/mapper/vg_iscsi-lv_client /media/data_iscsi/
```

## NFS

*server*

```shell
yum install -y nfs-utils nfs4-acl-tools
nmcli general hostname nfs.labs.local
systemctl restart systemd-hostnamed
cat > /etc/hosts <<eof
192.168.1.11 client.labs.local client
eof
systemctl restart network.service
systemctl list-unit-files | grep rpcbind
systemctl enable rpcbind.service
systemctl start rpcbind.service 
systemctl enable nfs-server.service 
systemctl start nfs-server.service
vim /etc/idmapd.conf 
:%s/#Domain = local.domain.edu/Domain = labs.local
ZZ
systemctl restart nfs
firewall-cmd --add-service=nfs --permanent 
firewall-cmd --reload
```

*client*

```shell
yum install -y nfs-utils nfs4-acl-tools 
nmcli general hostname client.labs.local
systemctl restart systemd-hostnamed
cat > /etc/hosts <<eof
192.168.1.10 nfs.labs.local nfs
eof
systemctl restart network.service
systemctl enable rpcbind.service
systemctl start rpcbind.service 
```

*share a dir*

*server*

```shell
mkdir /shome
chmod 0777 /shome
yum install -y policycoreutils-python
semanage fcontext -a -t public_content_rw_t "/shome(/.*)?"
restorecon -R /shome
echo "/shome    192.168.1.10/24(rw,sync,no_root_squash,no_subtree_check)" > /etc/exports
exportfs -avr
systemctl restart nfs-server
showmount -e localhost
```

*client*

```shell
mkdir /media/shome
mount -t nfs 192.168.1.10:/shome /media/shome
# permanent
echo "192.168.1.10:/shome   /media/shome    nfs rw,sync,hard,intr 0 0" >> /etc/fstab
mount
```

*autofs*

*server*

```shell
mkdir -p /home/dojomi/{dir01,dir02}
echo "/home/dojomi  192.168.1.10/24(rw,sync,no_root_squash,no_subtree_check)" >> /etc/exports
exportfs -avr
```

*client*

```shell
yum install autofs -y
systemctl list-unit-files | grep autofs
systemctl enable autofs.service
systemctl start autofs.service
echo "/home   /etc/auto.home --timeout 600" >> /etc/auto.master
echo "* -ro,soft,intr   192.168.1.10:/home/&" >> /etc/auto.home
systemctl restart autofs
useradd dojomi
su dojomi
ls -la
```

## Samba

*server*

```shell
yum install -y samba
cp -ri /etc/samba/smb.conf /etc/samba/smb.conf.bak
systemctl enable smb.service nmb.service
systemctl start smb.service nmb.service
firewall-cmd --permanent --add-service=samba
firewall-cmd --reload
cat > /etc/samba/smb.conf <<eof
[global]
    server string = %h *server* (Samba %v)
    workgroup = WORKGROUP
    netbios name = smb_svr
    interface = 192.168.1.10/24
    hosts allow = 127. 192.168.1.
    max protocol = SMB2
    log file = /var/log/samba-log.%m
    max log size = 1000
    max connections = 30
    invalid users = root 
    socket optinions = TCP_NODELAY
    netbios name = SHARE
    security = user
    passdb backend = tdbsam
    load printers = yes
    cups options = raw
    unix extensions = yes
    unix charset = UTF-8
    dos charset = CP932 
    encrypt passwords = true
    name resolv order = bcast host
    dns proxy = no
    #Guest
    #without username and pwd
    #public = yes
    #guest ok = no
    #every user who has an system acc but no samba acc has guest access
    map to guest = Bad User
    #user nobody has access
    #guest account = nobody
    time server = yes 
    #outsourcing to file
    #vim /etc/samba/smbshared.conf
eof
testpar
smbstatus
```

set printer and cdrom for all users

```shell
cat >> /etc/samba/smb.conf <<eof
[printers]
    comment = Printers
    browseable = no
    path = /var/spool/samba
    user client driver = yes
    printable = yes
    public = yes
    writable = no
    create mode = 0700
[cdrom]
    comment = CD/DVD
    read only = yes
    locking = no
    path = /cdrom
    writable = no
    guest ok = yes
eof
systemctl restart smb nmb
```

access to user home

```shell
useradd -d /home/dojomi dojomi
smbpasswd -a dojomi
cat >> /etc/samba/smb.conf <<eof
[homes]
    comment = HomeDIR
    browseable = no 
    valid users = %S
    writeable = yes
    create mode = 0640
    directory mode = 0750
eof
systemctl restart smb nmb
```

*client*

```shell
yum install -y samba-client
smbclient -N -L //192.168.1.10
smbclient //192.168.1.10/dojomi -U dojomi
ls 
# NT_STATUS_ACCESS_DENIED listening \*
```

*server*

```shell
yum install -y policycoreutils-python
getsebool -a | grep samba
setsebool -P samba_enable_home_dirs on
```

public share for all users

```shell
mkdir /media/public
chmod 0777 /media/public
cat >> /etc/samba/smb.conf <<eof
[public]
    comment = PublicDIR
    path = /media/public/
    public = yes
    browseable = yes
    writeable = yes 
    # no=NT_STATUS_MEDIA_WRITE_PROTECTED
eof
systemctl restart smb nmb
```

*client*

```shell
smbclient -N -L //192.168.1.10
smbclient //192.168.1.10/public
ls 
mkdir new
# NS_STATUS_ACCESS_DENIED making remote directory 
```

*server*

```shell
ls -ldZ /media/public/
semanage fcontext -a -t samba_share_t "/media/public(/.*)?" 
restorecon -R /media/public
systemctl restart smb nmb
```

share specific folder for group

```shell
mkdir /home/music
chmod 0770 /home/music
cat >> /etc/samba/smb.conf <<eof
[music]
    comment = MusicDIR
    path = /home/music
    writable = yes
    create mask = 0765
    valid users = @users
    admin users = dojomi
eof
systemctl restart smb nmb
usermod -G users dojomi
useradd -d /home/bob -s /bin/bash -G users -u 10002 bob
smbpasswd -a bob
chmod 0777 /home/music
```

*client*

```shell
smbclient -N -L //192.168.1.10
smbclient //192.168.1.10/music -U bob
ls 
mkdir new
# NS_STATUS_ACCESS_DENIED making remote directory 
```

*server*

```shell
ls -ldZ /home/music
semanage fcontext -a -t samba_share_t "/home/music(/.*)?" 
restorecon -R /home/music
```

mount share with cifs

```shell
yum install -y cifs-utils
mkdir -p /media/samba/music
mount -t cifs //192.168.1.10/music /media/samba/public -o user=dojomi
```

mount \@boot

```shell
mkdir /home/samba
echo “username=dojomi” > /home/samba/.smbcredentials
echo “password=samba_password” >> /home/samba/.smbcredentials
chmod 600 /media/samba/.smbcredentials

echo “//192.168.0.10/gacanepa /media/samba cifs credentials=/media/samba/.smbcredentials,defaults 0 0" >> /etc/fstab
mount -a
```

For configuration file
<https://github.com/DoJoMi/centbox_files/tree/master/samba>
