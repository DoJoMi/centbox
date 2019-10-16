## DHCP

------------------------------------------------------------------------

## Basic Usage

DNS (ns-server.labs.local)

```shell
yum install -y dhcp
echo -e "192.168.1.100 ns-server.labs.local ns" >> /etc/hosts
cp -p /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.bak
cp -p /usr/share/doc/dhcp-4.2.5/dhcpd.conf.example /etc/dhcp/dhcpd.conf
cat > /etc/dhcp/dhcpd.conf <<EOF
option domain-name "labs.local"
option domain-name-servers ns-server.labs.local; 
default-release-time 600;
max-lease-time 7200;
authoritative;
log-facility local7;
subnet 192.168.1.0 netmask 255.255.255.0 {
 range 192.168.1.15 192.168.1.20;
 option domain-name-servers ns-server.labs.local;
 option domain-name "labs.local";
 option routers 192.168.1.1;
 option broadcast-address 192.168.1.255;
 default-lease-time 3600;
 max-lease-time 7200;
}
# give host a special IP
host centos {
 hardware ethernet 08:00:27:B6:6F:72;
 fixed-address 192.168.1.19;
}
EOF
systemctl enable dhcpd; systemclt start dhcpd
tail -f /var/lib/dhcpd/dhcpd.lease /var/log/messages
Jun 18 18:24:05 localhost systemd: Started DHCPv4 Server Daemon.
Jun 18 18:27:33 localhost dhcpd: DHCPDISCOVER from 08:00:27:b6:6f:72 via enp0s3
Jun 18 18:27:33 localhost dhcpd: DHCPOFFER on 192.168.1.19 to 08:00:27:b6:6f:72 via enp0s3
Jun 18 18:27:33 localhost dhcpd: Dynamic and static leases present for 192.168.1.19.
Jun 18 18:27:33 localhost dhcpd: Remove host declaration centos or remove 192.168.1.19
Jun 18 18:27:33 localhost dhcpd: from the dynamic address pool for 192.168.1.0/24
Jun 18 18:27:33 localhost dhcpd: DHCPREQUEST for 192.168.1.19 (192.168.1.11) from 08:00:27:b6:6f:72 via enp0s3
Jun 18 18:27:33 localhost dhcpd: DHCPACK on 192.168.1.19 to 08:00:27:b6:6f:72 via enp0s3
```
