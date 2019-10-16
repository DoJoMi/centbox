# Host-Configurations

------------------------------------------------------------------------

hostname

```shell
nmcli general hostname server.labs.local
systemctl restart systemd-hostnamed
cat > /etc/hosts <<eof
192.168.1.10 server.labs.local server
eof 
systemctl restart network.service
```

locale language

```shell
# add for german de_DE.UTF-8
localectl status
yum install -y system-config-language
system-config-language
# manual
cat >/etc/locale.conf <<eof
LANG=en_US.utf8
LC_NUMERIC=en_US.UTF-8
LC_TIME=en_US.UTF-8
LC_MONETARY=en_US.UTF-8
LC_PAPER=en_US.UTF-8
LC_MEASUREMENT=en_US.UTF-8
eof
```

keyboard layout

```shell
yum insall -y system-config-keyboard
system-config-keyboard
# or
loadkeys de #only work until next reboot
setxkbmap de
# or
localectl list-keymaps | grep de-latin1
localectl set-keymap de-latin1-nodeadkeys
```

static IP-address

```shell
cat >/etc/sysconfig/network-scripts/ifcfg-enp0s3 <<eof
TYPE=Ethernet
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
NAME=enp0s3
UUID=350097ac-fa6f-438b-b28f-55dca0b2ef81
DEVICE=enp0s3
ONBOOT=yes
PEERDNS=yes
PEERROUTES=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPADDR=192.168.1.10
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
NETWORK=192.168.1.1
DNS1=8.8.8.8
eof
systecmtl restart network.service
```

ip forwarding

```shell
# needed if use for gateway/vpn/dial-in
cat > /etc/sysctl.conf <<eof
net.ipv4.ip_forward = 1
eof
sysctl -p
cat /proc/sys/net/ipv4/ip_forward
```

disable ipv6 if it\'s not used

```shell
cat > /etc/sysctl.conf << eof
# add to the  last line
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1 
eof
sysctl -p
```

bash

```shell
# open at login but not graphical (f.e programs,environment variables)
~/.profile || ~/.bash_profile(only bash)
# every time open terminal window (xterm) (f.e alias, functions)
~/.bashrc
```

```shell
# everytime you login call .bashrc
cat > ~./bash_profife <<eof
[ -f ~/.bashrc ] && source ~/.bashrc
[ -f ~/.profile ] && source ~/.profile
eof
```

**For more configuration details** <https://github.com/DoJoMi/dotbash>
