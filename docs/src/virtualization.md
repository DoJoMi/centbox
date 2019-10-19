# Virtualization

------------------------------------------------------------------------

## VirtualBox

```shell
wget http://centos.digitalnova.at/7.0.1406/isos/x86_64/CentOS-7.0-1406-x86_64-Everything.iso
yum remove -y VirtualBox-4*
cd /etc/yum.repos.d/ && wget http://download.virtualbox.org/virtualbox/rpm/rhel/virtualbox.repo
yum install -y binutils qt gcc make patch libgomp glibc-headers glibc-devel kernel-headers kernel-devel dkms
service vboxdrv start && virtualbox &
```

**Install a guest OS via Kickstart file see installation details**
<https://github.com/DoJoMi/ks_vbox>

## KVM

```shell
# delete default virtbr0 
brctl show
nmcli con show
nmcli con del virbr0
ifconfig virbr0 down
brctl delbr virbr0

# generate a new network UUID - /proc/sys/kernel/random/uuid
uuidgen enp3s0 #--> e6b60243-e383-4566-9e66-aebf18628cda

# set default connection enp3s0 to a static address
cat > /etc/sysconfig/network-scripts/ifcfg-enp3s0 << eof
TYPE=Ethernet 
BOOTPROTO=none 
NM_CONROLLED=no
NAME=enp3s0
UUID=e6b60243-e383-4566-9e66-aebf18628cda
DEVICE=enp3s0
ONBOOT=yes
IPADDR=192.168.1.7
PREFIX=24
GATEWAY=192.168.1.1
DNS1=192.168.0.1
DNS2=8.8.8.8
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no                                                                     
BRIDGE=kvm
eof

# setup a new virual bridge named kvm
cat > /etc/sysconfig/network-scripts/ifcfg-kvm << eof
NAME=kvm
DEVICE=kvm
TYPE=Bridge
BOOTPROTO=static
ONBOOT=yes
STP=no
DELAY=0
IPADDR=192.168.1.10
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
DNS1=192.168.0.1                                                                
DNS2=8.8.8.8
eof

echo "BRIDGE=kvm" >> /etc/sysconfig/network-scripts/ifcfg-enp3s0
systemctl restart network.service

nmcli con show
NAME    UUID                                  TYPE            DEVICE 
kvm     6c97e217-58ad-b10f-5b30-9aad04cf8be3  bridge          kvm    
enp3s0  e6b60243-e383-4566-9e66-aebf18628cda  802-3-ethernet  enp3s0 
```

```shell
# install virtual-manager & check support
egrep -c '(vmx|svm)' /proc/cpuinfo
# 2
yum install -y groupinstall "Virtualization Tools" 
yum install -y qemu-kvm libvirt virt-install bridge-utils qemu-kvm-tools
systemctl start libvirtd
systemctl enable libvirtd

virsh -c qemu:///system list
# if an error returns add the next 2lines
adduser dojomi libvirtd
adduser dojomi kvm
logout
```

```shell
# create a new vm_image with name centos under location /var/lib/libvirt/images/
qemu-img create -f qcow2 -o preallocation=metadata /var/lib/libvirt/images/centos.qcow2 11G

virt-install \
--name centos \
--ram 1024 \
--disk path= /var/lib/libvirt/images/centos.qcow2.qcow2,size=11 \
--vcpus 1 \
--os-type linux \
--os-variant rhel7 \
--network bridge=kvm \
--graphics none \
--console pty,target_type=serial \
--location 'http://mirror.easyname.at/centos/7/os/x86_64/' \
--extra-args 'console=ttyS0,115200n8 serial'
```

**Install guest OS via virt-install using Kickstart file**
<https://github.com/DoJoMi/ks_virt>

```shell
# show all installed vm 
virsh --connect qemu:///system list --all
# start a specific vm
virsh --connect qemu:///system start centos
# shutdown vm
virsh --connect qemu:///system shutdown centos
# get information about specific vm
virsh --connect qemu:///system dominfo centos   
# console
virsh --connect qemu:///system console
virsh --connect qemu:///system start centos --console

# VNC 
# --> if inlcuded inside script --graphics vnc,listen=0.0.0.0 --noautoconsole \
virsh --connect qemu:///system vncdisplay centos
#--> or deeper into xml file
virsh --connect qemu:///system edit centos
# <graphics type='vnc' port='-1' autoport='yes' keymap='de'/>
virsh --connect qemu:///system start centos --console
firewall-cmd --add-port=5900/tcp --permanent
firewall-cmd --reload
virsh --connect qemu:///system vncdisplay centos
# :0 means 5900, :1 5901
# connect to it with
vncviewer localhost:0 -geometry 1024x768
vnc://<IP>:<PORT>

# remove
virsh --connect qemu:///system destroy centos
virsh --connect qemu:///system undefine centos
# delete vm-image
rm /var/lib/libvirt/images/centos.qcow2
```

## LXC

```shell
yum install -y wget epel-release 
yum install -y lxc lxc-templates libvirt openssh-server
systemctl start libvirtd
lxc-create -n lxc_vm -t download
# Distribution: centos
# Release: 7
# Architecture: amd64
chroot /var/lib/lxc/lxc_vm/rootfs passwd
lxc-ls -f
lxc-start -d -n lxc_vm
# start on a specific tty
lxc-start -n <name> -d -c /dev/tty4
lxc-attach -n lxc_vm
# high cpu laod systemd-journal
exit
echo -e "lxc.autodev = 1" >> /var/lib/lxc/lxc_vm/config
echo -e "lxc.kmsg = 0" >> /var/lib/lxc/lxc_vm/config
sed -i '24s/^/#/' /var/lib/lxc/lxc_vm/rootfs/lib/systemd/system/getty@.service
lxc-stop -n lxc_vm
lxc-start -d -n lxc_vm
lxc-attach -n lxc_vm
# not longer needed
lxc-destroy -n lxc_vm
```

```shell
# LXC-Web-Panel
# installation script not working on centos
git clone https://github.com/lxc-webpanel/LXC-Web-Panel.git
mv LXC-Web-Panel/ /srv/lwp
yum install -y python python-pip
pip install --upgrade pip
pip install flask==0.9
# change the standart port
vim /srv/lwp/lwp.conf
yum install -y httpd
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-port=5000/tcp
firewall-cmd --reload
cd /srv/lwp
python lwp.py 
```
