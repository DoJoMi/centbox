# DNS

------------------------------------------------------------------------

## BIND

```shell
nmcli general hostname pmaster.labs.local
systemctl restart systemd-hostnamed
echo "192.168.1.100 ns-server.labs.local dns" >> /etc/hosts
yum install -y bind bind-utils
firewall-cmd --permanent --add-service dns
firewall-cmd --reload
yum install -y ntp
sed -i "s/^#restrict/restrict/" /etc/ntp.conf
cat >> /etc/ntp.conf <<EOF
logfile /var/log/ntp.log
EOF
firewall-cmd --add-service=ntp --permanent
firewall-cmd --reload
systemctl enable ntpd.service; systemctl start ntpd.service
ntpq -p
ntpdate || date
cp -rv /etc/named.conf /etc/named.conf.bak
cp -rv /var/named/named.localhost /var/named/fwdz-labs.local
cp -rv /var/named/named.loopback /var/named/rvz-labs.local
cat >> /etc/named.conf <<EOF
zone "ns-server.labs.local" IN {
    type master;
    file "fwdz-labs.local"; #file is located in /var/named
    allow-update { none; };
};
zone "1.168.192.in-addr.arpa" IN {
    notify no;
    type master;
    file "rvz-labs.local"; #file is located in /var/named
    allow-update { none; };
};
EOF
named-checkconf /etc/named.conf
cat > /var/named/fwdz-labs.local <<EOF
$TTL 1D
@   IN SOA  ns-server.labs.local. root.labs.local.(
                    0   ; serial
                    1D  ; refresh
                    1H  ; retry
                    1W  ; expire
                    3H )    ; minimum
@ IN NS        ns-server.labs.local.
@ IN  A        192.168.1.100
@ IN  MX 10    mail-server.labs.local.

ns-server   IN  A 192.168.1.100
mail-server IN  A 192.168.1.11
web-server  IN  A 192.168.1.12

mail    IN  CNAME   mail-server.labs.local.
www     IN  CNAME   web-server.labs.local.
EOF
chown root:named /var/named/fwdz-labs.local
named-checkzone labs.local /var/named/fwdz-labs.local
cat > /var/named/rvz-labs.local <<EOF
$TTL 1D
@   IN SOA  ns-server.labs.local. root.labs.local.(
                    0   ; serial
                    1D  ; refresh
                    1H  ; retry
                    1W  ; expire
                    3H )    ; minimum
@  IN  NS  ns-server.labs.local.
@  IN  PTR labs.local.
@  IN  A   255.255.255.0

100 IN  PTR ns-server.labs.local.
11 IN  PTR mail-server.labs.local.
12 IN  PTR web-server.labs.local.
EOF
chown root:named /var/named/rvz-labs.local
named-checkzone 1.168.192-addr.arpa /var/named/rvz-labs.local
systemctl restart named
# quick test
sed -i "s/^nameserver=8.8.8.8/nameserver=192.168.1.100/" /etc/resolv.conf
ping google.com
dig @localhost ns-server.labs.local
nslookup ns-server.labs.local
Server:     192.168.1.100
Address:    192.168.1.100#53

Name:   ns-server.labs.local
Address: 192.168.1.100
```

## Unbound

Caching only DNS

server

```shell
yum install -y unbound
systemctl enable unbound; systemctl start unbound
nmcli general hostname dns.labs.local
systemctl restart systemd-hostnamed
cat > /etc/unbound/unbound.conf <<EOF
server:
    verbosity: 1
    statistics-interval: 0
    statistics-cumulative: no
    extended-statistics: yes
    num-threads: 2
    interface-automatic: no
    interface: 192.168.1.11
    logfile: /var/log/unbound
    hide-identity: yes
    hide-version: yes
    domain-insecure: "labs.local"
    access-control: 192.168.1.0/24 allow
    chroot: ""
    username: "unbound"
    directory: "/etc/unbound"
    log-time-ascii: yes
    pidfile: "/var/run/unbound/unbound.pid"
    harden-glue: yes
    harden-dnssec-stripped: yes
    harden-below-nxdomain: yes
    harden-referral-path: yes
    use-caps-for-id: no
    unwanted-reply-threshold: 10000000
    prefetch: yes
    prefetch-key: yes
    rrset-roundrobin: yes
    minimal-responses: yes
    trusted-keys-file: /etc/unbound/keys.d/*.key
    auto-trust-anchor-file: "/var/lib/unbound/root.key"
    val-clean-additional: yes
    val-permissive-mode: no
    val-log-level: 1
    include: /etc/unbound/local.d/*.conf
remote-control:
    control-enable: yes
    server-key-file: "/etc/unbound/unbound_server.key"
    server-cert-file: "/etc/unbound/unbound_server.pem"
    control-key-file: "/etc/unbound/unbound_control.key"
    control-cert-file: "/etc/unbound/unbound_control.pem"
include: /etc/unbound/conf.d/*.conf

forward-zone:
    name:"."
    forward-addr: 8.8.8.8
    forward-addr: 8.8.4.4
EOF
unbound-checkconf
systemctl restart unbound.service
firewall-cmd --permantent -add-service=dns
firewall-cmd --reload
unbound-control status
unbound-control list_forwards
```

client

```shell
nmcli general hostname client.labs.local
systemctl restart systemd-hostnamed
cat >> /etc/hosts <<EOF
192.168.1.11 dns.labs.local dnssrv
EOF
cat >> /etc/sysconfig/network-scripts/ifcfg-enp0s3 <<EOF
DNS1=192.168.1.11
EOF
yum install -y bind-utils || yum install -y ldns
dig example.com || host -a example.com || drill example.com 
drill labs.local
# ;; ->>HEADER<<- opcode: QUERY, rcode: NXDOMAIN, id: 28174
# ;; flags: qr rd ra ; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 0 
# ;; QUESTION SECTION:
# ;; labs.local.    IN  A
#
# ;; ANSWER SECTION:
#
# ;; AUTHORITY SECTION:
# . 1798    IN  SOA a.root-servers.net. nstld.verisign-grs.com. 2016052500 1800 900 604800 86400
#
# ;; ADDITIONAL SECTION:
#
# ;; Query time: 35 msec
# ;; SERVER: 192.168.1.11
# ;; WHEN: Wed May 25 10:54:31 2016
# ;; MSG SIZE  rcvd: 103
```
