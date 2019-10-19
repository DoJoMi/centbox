# Kernel

------------------------------------------------------------------------

## Compile Kernel

```shell
# kernel versions
uname -a #--> kernel information
cat/proc/cpuinfo
# x86_64    #--> 64 bit kernel
# i686      #--> 32 bit kernel
```

```shell
# runtime kernel modules
lsmod    #--> modules which are on rdy status
modprobe #--> add/remove kernel modules
# get information of specific module
modinfo /lib/modules/$(uname -r)/kernel/drivers/usb/storage/usb-storage.ko
# integrate a module
insmod /lib/modules/$(uname -r)/kernel/drivers/usb/storage/usb-storage.ko
lsmod | grep usb_storage
# remove a module which is not used
cat /proc/modules | grep usb
rmmod usb-storage.ko
```

```shell
# compile new kernel
ls -la /usr/src #--> dynamic source files
ls -la /boot    #--> static source files
```

```shell
# install relevant packages
yum install -y make ncurses-devel wget gcc dracut patch openssl-devel bc
```

```shell
# get latest kernel
wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.5.tar.xz /usr/src/
cd /usr/src/ && tar -xf linux-4.5.tar.xz
ln -s linux-4.5 linux
```

```shell
# f.e patch it with BFS (very low overheard CPU scheduler) (kernel v.4.3)
# !!PATCH NOT TESTED!!
wget http://ck.kolivas.org/patches/bfs/4.0/4.5/4.5-sched-bfs-469.patch
cp 4.5-sched-bfs-469.patch linux/ && cd linux/
# selftest
patch -p1 --dry-run < 4.5-sched-bfs-469.patch
# patch it
patch -p1 < 4.5-sched-bfs-469.patch 
```

```shell
# compile & install it
# make clean: will remove all .o and .ko
make clean
# make mrproper: make clean + remove config dependencies and "make config" creations
# copy the current config
cp /boot/config-`uname -r` /usr/src/linux/.config
make menuconfig
# <Load>
# .config 
# [*] = support inside the kernel
# <M> = not loaded directly
#  N = uncheck
```

```shell
# standard build process (manual)
# -j3 for 2core
make -j3
make modules_install
make install
# it creates files into /boot 
# vmlinuz-$(uname -r)    –-> kernel
# System.map-$(uname -r) –-> exported kernel symbols 
# initrd.img-$(uname -r) –-> initrd image (relevant boot time files)
# config-$(uname -r)     –-> conf file
```

```shell
# update grub2
grub2-mkconfig -o /boot/grub2/grub.cfg
```

```shell
# to take the new compiled kernel with you
make binrpm-pkg
```

```shell
# remove the older kernels
rpm -q kernel
yum install -y yum-utils
package-cleanup --oldkernels --count=1
# permanent /etc/yum/yum.conf under installonly_limit=
```

```shell
# create your own kernel module
yum install -y gcc make kernel-headers-$(uname -r)
git clone https://github.com/DoJoMi/k_mod.git
cd k_mod && make module
insmod hello_kernel.ko
dmegs | tail -1
rmmod hello_kernel.ko
dmegs | tail -1
```
