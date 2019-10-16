# Identity-Management

------------------------------------------------------------------------

## Kerberos

centvm1

```shell
echo "192.168.1.11 centvm1.labs.local centvm1" >> /etc/hosts
echo "192.168.1.12 centvm2.labs.local centvm2" >> /etc/hosts
#kerberos requires time synchronization
yum install -y ntp
sed -i "s/^#restrict/restrict/" /etc/ntp.conf
cat >> /etc/ntp.conf <<EOF
logfile /var/log/ntp.log
EOF
firewall-cmd --add-service=ntp --permanent
firewall-cmd --reload
systemctl enable ntpd.service; systemctl start ntpd.service
ntpq -p
```

centvm2

```shell
echo "192.168.1.11 centvm1.labs.local centvm1" >> /etc/hosts
echo "192.168.1.12 centvm2.labs.local centvm2" >> /etc/hosts
```

centvm1

```shell
yum install -y krb5-server krb5-libs
cp -rv /etc/krb5.conf /etc/krb5.conf.bak
cat > /etc/krb5.conf <<EOF
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
 default_realm = LABS.LOCAL
 default_ccache_name = KEYRING:persistent:%{uid}

[realms]
 LABS.LOCAL = {
  kdc = centvm1.labs.local
  admin_server = centvm1.labs.local
 }

[domain_realm]
 .labs.local = LABS.LOCAL
 labs.local= LABS.LOCAL 
EOF
cat > /var/kerberos/krb5kdc/kdc.conf << EOF
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 LABS.LOCAL = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }
EOF
cat > /var/kerberos/krb5kdc/kadm5.acl <<EOF
*/admin@LABS.LOCAL  *
EOF
kdb5_util create -s -r LABS.LOCAL
# Loading random data
# nothing happens
yum provides */ngnd
yum install -y rng-tools
cat > /usr/lib/systemd/rngd.service <<EOF
[Unit]
Description=Hardware RNG Entropy Gatherer Daemon
[Service]
ExecStart=/sbin/rngd -f -r /dev/urandom
[Install]
WantedBy=multi-user.target
EOF
cp -rv /usr/lib/systemd/rngd.service /etc/systemd/system
systemctl daemon-reload
systemctl start rngd.service
# reuse it
kdb5_util create -s -r LABS.LOCAL
systemctl enable krb5kdc kadmin 
systemctl start krb5kdc.service kadmin.service
firewall-cmd --get-services | grep kerberos --color
firewall-cmd --permanent --add-service kerberos 
firewall-cmd --reload
useradd dojomi
kadmin.local
addprinc root/admin
addprinc dojomi
addprinc -randkey host/centvm2.labs.local
ktadd -k /tmp/centvm2.labs.local host/centvm2.labs.local
listprincs
quit
scp /etc/krb5.conf centvm2:/etc/krb5.conf
scp /tmp/centvm2.labs.local centvm2:/tmp/centvm2.labs.local
```

centvm2

```shell
yum install -y pam_krb5 krb5-workstation
ktutil
rkt /tmp/centvm2.labs.local
wkt /etc/krb5.keytab
quit
kinit dojomi
klist
```

For configuration files
<https://github.com/DoJoMi/centbox_files/tree/master/kerberos>
