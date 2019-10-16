# Security

------------------------------------------------------------------------

## ClamAV

```shell
yum install -y clamav clamav-freshclam && freshclam
```

```shell
cat > /home/$USER/clamav.sh <<eof
#!/bin/bash

DATE=`date "+%Y%m%d"`
SCAN_DIR=$1 # /home/$USER/Desktop
LOG_FILE=$2 # /home/$USER/clamav.log
MAIL=$3

/usr/bin/clamscan -ir $SCAN_DIR | tail | grep 'Infected' > $LOG_FILE
mail -s "ClamAV-Log $DATE " $MAIL < $LOG_FILE
rm $LOG_FILE
eof
```

```shell
crontab -e
0 3 * * 1-5 /home/$USER/clamav.sh
```

## Fail2ban

```shell
yum install -y epel-release && yum install -y fail2ban
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
cat > /etc/fail2ban/jail.local <<eof
[DEFAULT]
# Ban hosts for one hour:
bantime = 3600
[sshd]
enabled = true
eof
cat > /etc/fail2ban/jail.d/sshd.local <<eof
[ssh-iptables]
enabled  = true
filter   = sshd
action   = iptables[name=SSH, port=ssh, protocol=tcp]
logpath  = %(sshd_log)s
maxretry = 3
bantime = 86400
eof
systemctl enable fail2ban && systemctl start fail2ban
```

```shell
yum install -y denyhosts
systemctl enable denyhosts
cat > /etc/hosts.allow <<eof
sshd: IP [192.168.1.]
ALL : IP [192.168.1.]
ALL : 127.0.0.1
eof   
cat > /etc/hosts.deny <<eof
sshd: ALL
eof
```

fail2ban-client

```shell
fail2ban-client -i
fail2ban> status ssh
fail2ban> set ssh unbanip 195.94.185.239
```

## Iptables

save rules

```shell
iptables-save > /etc/iptables/rules.v4
```

flush all rules and set default chain

```shell
iptables -F
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT DROP
```

show ruleset

```shell
iptables -L -v
iptables -L INPUT -v
iptables -L -v -n
```

block specific addresses

```shell
BLOCK_THIS_IP="x.x.x.x"
iptables -A INPUT -s "$BLOCK_THIS_IP" -j DROP
iptables -A INPUT -i eth0 -s "$BLOCK_THIS_IP" -j DROP
iptables -A INPUT -i eth0 -p tcp -s "$BLOCK_THIS_IP" -j DROP
```

allow incoming traffic

```shell
iptables -A INPUT -i eth0 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
```

allow incoming from specific network

```shell
iptables -A INPUT -i eth0 -p tcp -s 192.168.100.0/24 --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
```

allow outgoing

```shell
iptables -A OUTPUT -o eth0 -p tcp -d 192.168.100.0/24 --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
```

combine ruleset

```shell
iptables -A INPUT -i eth0 -p tcp -m multiport --dports 22,443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp -m multiport --sports 22,443 -m state --state ESTABLISHED -j ACCEPT
```

loadbalancing traffic

```shell
iptables -A PREROUTING -i eth0 -p tcp --dport 443 -m state --state NEW -m nth --counter 0 --every 3 --packet 0 -j DNAT --to-destination 192.168.1.101:443
iptables -A PREROUTING -i eth0 -p tcp --dport 443 -m state --state NEW -m nth --counter 0 --every 3 --packet 1 -j DNAT --to-destination 192.168.1.102:443
iptables -A PREROUTING -i eth0 -p tcp --dport 443 -m state --state NEW -m nth --counter 0 --every 3 --packet 2 -j DNAT --to-destination 192.168.1.103:443
```

## VPN

### wireguard

centvm1:

```shell
yum update -y && yum upgrade -y
curl -Lo /etc/yum.repos.d/wireguard.repo https://copr.fedorainfracloud.org/coprs/jdoss/wireguard/repo/epel-7/jdoss-wireguard-epel-7.repo
yum install -y epel-release && yum install wireguard-dkms wireguard-tools -y
reboot
ip a
# 3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
# link/ether 0a:f5:dd:80:04:75 brd ff:ff:ff:ff:ff:ff
# inet 10.135.30.145/16 brd 10.135.255.255 scope global eth1
#    valid_lft forever preferred_lft forever
wg genkey > privkey
umask 077 privkey
wg pubkey < private > pubkey
ip link add dev wg0 type wireguard
ip address add dev wg0 192.168.0.1/24
wg set wg0 private-key ./privkey
ip link set wg0 up
ip a
# 4: wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
# link/none 
# inet 192.168.0.1/24 scope global wg0
#   valid_lft forever preferred_lft forever
wg
# interface: wg0
# public key: TeBCqKx0JpYTUyxIB1NTcQoA1aIkWawgUccyqK56PDo=
# private key: (hidden)
# listening port: 48028
wg set wg0 peer 8MlMjAyA75IkdSRLbgxUHE4UBcPZ9qf1xGgDxhVYtQ8= allowed-ips 192.168.0.2/32 endpoint 10.135.32.250:58977
```

centvm2:

```shell
yum update -y && yum upgrade -y
curl -Lo /etc/yum.repos.d/wireguard.repo https://copr.fedorainfracloud.org/coprs/jdoss/wireguard/repo/epel-7/jdoss-wireguard-epel-7.repo
yum install -y epel-release && yum install wireguard-dkms wireguard-tools
reboot
ip a
# 3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
# link/ether 52:de:b5:f5:bd:c7 brd ff:ff:ff:ff:ff:ff
# inet 10.135.32.250/16 brd 10.135.255.255 scope global eth1
#   valid_lft forever preferred_lft forever
wg genkey > privkey
umask 077 privkey
wg pubkey < privkey > pubkey
ip link add dev wg0 type wireguard
ip address add dev wg0 192.168.0.2/24
wg set wg0 private-key ./privkey
ip link set wg0 up
ip a
# 4: wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
# link/none 
# inet 192.168.0.2/24 scope global wg0
#   valid_lft forever preferred_lft forever
wg
# interface: wg0
# public key: 8MlMjAyA75IkdSRLbgxUHE4UBcPZ9qf1xGgDxhVYtQ8=
# private key: (hidden)
# listening port: 58977
wg set wg0 peer TeBCqKx0JpYTUyxIB1NTcQoA1aIkWawgUccyqK56PDo= allowed-ips 192.168.0.1/32 endpoint 10.135.30.145:48028
ping 192.168.0.1
# PING 192.168.0.1 (192.168.0.1) 56(84) bytes of data.
# 64 bytes from 192.168.0.1: icmp_seq=1 ttl=64 time=5.34 ms
wg
# interface: wg0
# public key: 8MlMjAyA75IkdSRLbgxUHE4UBcPZ9qf1xGgDxhVYtQ8=
# private key: (hidden)
# listening port: 58977
# peer: TeBCqKx0JpYTUyxIB1NTcQoA1aIkWawgUccyqK56PDo=
# endpoint: 10.135.30.145:48028
# allowed ips: 192.168.0.1/32
# latest handshake: 10 minutes, 53 seconds ago
# transfer: 564 B received, 476 B sent
```

### openvpn

vpn-network : 10.8.0.0/24

```shell
yum update -y && yum install epel-release -y && yum install -y openvpn easy-rsa firewalld
systemctl enable firewalld && systemctl start firewalld
cat > /etc/openvpn/server.conf <<eof
dev tun
proto udp
port 1194
keepalive 10 120
max-clients 5

# certs
ca ca.crt
cert server.crt
key server.key
dh dh.pem
tls-auth ta.key 0

# cipher
reneg-sec 0
remote-cert-tls client
crl-verify crl.pem
tls-version-min 1.2
cipher AES-128-CBC
auth SHA256
tls-cipher TLS-DHE-RSA-WITH-AES-256-GCM-SHA384:TLS-DHE-RSA-WITH-AES-256-CBC-SHA256:TLS-DHE-RSA-WITH-AES-128-GCM-SHA256:TLS-DHE-RSA-WITH-AES-128-CBC-SHA256

# privileges
user nobody
group nobody

# pool
server 10.8.0.0 255.255.255.0
topology subnet
ifconfig-pool-persist ipp.txt
client-config-dir ccd

persist-key
persist-tun
;comp-noadapt

# force all traffic through vpn
push "redirect-gateway def1 bypass-dhcp"
# to connect to the lan of server from client a route is needed
;push "route 192.168.1.0 255.255.255.0"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"

# log
log-append /var/log/openvpn.log
verb 4
eof
```

```shell
mkdir /etc/openvpn/ccd
cat > /etc/openvpn/ccd/vpn-client-01 <<eof
# MULTI: bad source address from client [IP ADDRESS], packet dropped
# iroute is only to tell the server at which client it should send traffic
iroute 192.168.1.0 255.255.255.0
# give the client always the same ip
ifconfig-push 10.8.0.2 255.255.255.0
eof
```

```shell
/usr/share/easy-rsa/3/easyrsa init-pki
/usr/share/easy-rsa/3/easyrsa build-ca nopass
# ENTER
/usr/share/easy-rsa/3/easyrsa gen-dh
/usr/share/easy-rsa/3/easyrsa build-server-full vpn-server nopass
/usr/share/easy-rsa/3/easyrsa build-client-full vpn-client-01 nopass
/usr/share/easy-rsa/3/easyrsa gen-crl
openvpn --genkey --secret pki/ta.key
```

```shell
cp pki/ca.crt /etc/openvpn/ca.crt
cp pki/dh.pem /etc/openvpn/dh.pem
cp pki/issued/vpn-server.crt /etc/openvpn/server.crt
cp pki/private/vpn-server.key /etc/openvpn/server.key
cp pki/ta.key /etc/openvpn/ta.key
cp pki/crl.pem /etc/openvpn/crl.pem
```

```shell
cat > /etc/sysctl.conf <<eof 
net.ipv4.ip_forward = 1 
eof
sysctl -p
firewall-cmd --add-service openvpn --permanent
firewall-cmd --add-masquerade --permanent 
interface=$(ip route get 8.8.8.8 | awk 'NR==1 {print $(NF-2)}')
firewall-cmd --direct --passthrough ipv4 -t nat -A POSTROUTING -s 10.8.0.0/24 -o $interface -j MASQUERADE
firewall-cmd -t nat -A POSTROUTING -s 10.8.0.0/24 -o $interface -j SNAT --to 46.101.147.109
firewall-cmd --reload
systemctl -f enable openvpn@server.service && systemctl start openvpn@server.service
```

```shell
mkdir vpn-client-01-config
cp pki/ca.crt vpn-client-01-config/ca.crt
cp pki/issued/vpn-client-01.crt vpn-client-01-config/client.crt
cp pki/private/vpn-client-01.key vpn-client-01-config/client.key
cp pki/ta.key vpn-client-01-config/ta.key

# add <IP-ADDRESS-OF-OPENVPN-SERVER> and Certificates
cat > vpn-client-01-config/client.ovpn <<eof
tls-client
pull
client
dev tun
proto udp
remote <IP-ADDRESS-OF-OPENVPN-SERVER> 1194
redirect-gateway def1
nobind
persist-key
persist-tun
;comp-noadapt
verb 3
ca ca.crt
cert client.crt
key client.key
tls-auth ta.key 1
remote-cert-tls server
key-direction 1
cipher AES-128-CBC
auth SHA256
tls-version-min 1.2
auth-nocache
tun-mtu 1500
eof
```

```shell
tar cvfz vpn-client-01-config.tgz vpn-client-01-config
```

client-side

```shell
yum install epel-release -y && yum install -y openvpn easy-rsa
scp <user>@<host>:vpn-client-01-config.tgz <location>
tar -xvzf /<location>/vpn-client-01-config.tgz
cd vpn-client-01-config
# ls -laZ
# semanage fcontext -a -t home_cert_t <files>
# restorecon -R -v <files>
# check your current external-ip
curl ipinfo.io/ip
openvpn --config client.ovpn
```

```shell
# remove on server if not longer needed
rm -rf vpn-client-01-config vpn-client-01-config.tgz
```
