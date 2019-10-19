# Mail

------------------------------------------------------------------------

## Postfix,Dovecot \[hash:BerkleyDB\],squirrelmail

```shell
# dns-configuration
# Forward zone for labs.local
IN MX 10    mail-server.labs.local.
mail-server.labs.local. IN A        192.168.1.11

# Reverse zone for labs.local
192.168.1.11        IN PTR  mail-server.labs.local.
```

```shell
# smtp
yum install -y postfix cyrus-* telnet
nmcli general hostname mail-server.labs.local
systemctl restart systemd-hostnamed
echo "104.248.251.121 mail.kubeops.rocks" >> /etc/hosts
    # echo "192.168.1.100 ns-server.labs.local ns" >> /etc/hosts
cp -rv /etc/postfix/main.cf /etc/postfix/main.cf.bak
cp -rv /etc/postfix/master.cf /etc/postfix/master.cf.bak
# create a new vuser
groupadd vpostfix -g 5000 && useradd vpostfix -r -g 5000 -u 5000 -s /sbin/nologin -c "Virtual postfix user" -d /var/empty
grep vpostfix /etc/passwd
# vpostfix:x:5000:5000:Virtual postfix user:/var/empty:/sbin/nologin
# vhost configuration
postconf -e 'virtual_mailbox_domains = /etc/postfix/virtual_domains'
postconf -e 'virtual_mailbox_base = /var/mail/vhosts'
postconf -e 'virtual_mailbox_maps = hash:/etc/postfix/vmailbox'
postconf -e 'virtual_minimum_uid = 1001'
postconf -e 'virtual_uid_maps = static:5000'
postconf -e 'virtual_gid_maps = static:5000'
postconf -e 'virtual_alias_maps = hash:/etc/postfix/virtual'
# main configuration
postconf -e 'myhostname= mail-server.labs.local'
postconf -e 'mydomain= labs.local'
postconf -e 'myorigin= $mydomain'
postconf -e 'inet_interfaces=all'
postconf -e 'mydestination = $myhostname, localhost.$mydomain, localhost'
postconf -e 'mynetworks = 127.0.0.0/8, 192.168.1.0/24'
postconf -e 'home_mailbox= Maildir/'
postconf -e 'message_size_limit = 10485760'
postconf -e 'mailbox_size_limit = 1073741824'
postconf -e 'smtpd_banner = $myhostname ESMTP $mail_name ($mail_version)'
# if it is activated just uncommend it  
# sed -i "s/inet_interfaces = localhost/#inet_interfaces = localhost/" /etc/postfix/main.cf
# create file for virtual_domains
echo "labs.local" > /etc/postfix/virtual_domains
# store mail for all vdomains
mkdir /var/mail/vhosts
chgrp -R vpostfix /var/mail
mkdir /var/mail/vhosts/labs.local
cd /var/mail
chown -R vpostfix:vpostfix vhosts 
# all mails are delivered uneder  /var/mail/vhosts/labs.local/<virtual-user>
# end up with / or mail will not delivered
cat > /etc/postfix/vmailbox <<eof
dojomi@labs.local        labs.local/dojomi/
admin@labs.local       labs.local/admin/
@labs.local           labs.local/all/
eof
# create virtual/local alias file   
touch /etc/postfix/virtual && cd /etc
postalias aliases
# execute anytime if u make changes inside virtual_domains vmailbox
postmap /etc/postfix/virtual
postmap /etc/postfix/vmailbox
systemctl enable postfix; systemctl start postfix
# if u want to check from outside
netstat -ntlp
# firewall-cmd --permanent --add-port 25/tcp
# firewall-cmd --reload

telnet localhost 25
Trying ::1...
Connected to localhost.
Escape character is '^]'.
220 mail-server.labs.local ESMTP Postfix (2.10.1)
ehlo localhost
250-mail-server.labs.local
250-PIPELINING
250-SIZE 10485760
250-VRFY
250-ETRN
250-AUTH PLAIN LOGIN
250-ENHANCEDSTATUSCODES
250-8BITMIME
250 DSN
quit
```

```shell
# pop3/imap
yum install -y dovecot
cp -rv /etc/dovecot/dovecot.conf /etc/dovecot/dovecot.conf.bak
cp -rv /etc/dovecot/conf.d/10-auth.conf /etc/dovecot/conf.d/10-auth.conf.bak
cp -rv /etc/dovecot/conf.d/10-mail.conf /etc/dovecot/conf.d/10-mail.conf.bak
cp -rv /etc/dovecot/conf.d/10-master.conf /etc/dovecot/conf.d/10-master.conf.bak
cp -rv /etc/dovecot/conf.d/10-ssl.conf /etc/dovecot/conf.d/10-ssl.conf.bak
cp -rv /etc/dovecot/conf.d/10-logging.conf /etc/dovecot/conf.d/10-logging.conf.bak
vim /etc/dovecot/dovecot.conf
protocols = imap pop3 lmtp 
listen = *
vim /etc/dovecot/conf.d/10-auth.conf
disable_plaintext_auth = no
auth_mechanisms = plain login
!include auth-passwdfile.conf.ext
vim /etc/dovecot/conf.d/10-logging.conf
log_path = /var/log/dovecot.log
auth_verbose = no
auth_debug = no
verbose_ssl = no
log_timestamp = "%Y-%m-%d %H:%M:%S "
vim /etc/dovecot/conf.d/10-mail.conf
mail_home = /var/mail/vhosts/%d/%n
mail_location = maildir:~
mail_uid = 5000    # These are the GID and UID numbers for vpostfix
mail_gid = 5000    # Don't just put random numbers here. Check above.
mail_privileged_group = vpostfix
vim /etc/dovecot/conf.d/10-master.conf
unix_listener auth-userdb {
  mode = 0600
  user = vpostfix
  group =  vpostfix
}
# Postfix smtp-auth
unix_listener /var/spool/postfix/private/auth {
  mode = 0666
  user = vpostfix
  group = vpostfix
}
vim /etc/dovecot/conf.d/10-ssl.conf
ssl = no
# ssl_cert = </etc/ssl/certs/dovecot.pem
# ssl_key = </etc/ssl/private/dovecot.pem
# add new virtual user included in /etc/dovecot/conf.d/10-auth.conf with !auth-passwdfile.conf.ext
# cat /etc/dovecot/conf.d/auth-passwdfile.conf.ext --> user stored in /etc/dovecot/users
doveadm pw -s SHA512-CRYPT >> /etc/dovecot/users
dojomi@lab.local:{SHA}::::
admin@lab.local:{SHA}::::
systemctl enable dovecot;systemctl start dovecot
netstat -ntlp
# firewall-cmd --permanent --add-port={110/tcp,143/tcp} 
# firewall-cmd --reload

telnet localhost 110
Trying ::1...
telnet: connect to address ::1: Connection refused
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
+OK Dovecot ready.
user dojomi@lbx.io
+OK
pass dojomi
+OK Logged in.
stat
+OK 0 0
quit
+OK Logging out.
Connection closed by foreign host.

telnet localhost 143
Trying ::1...
telnet: connect to address ::1: Connection refused
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
* OK [CAPABILITY IMAP4rev1 LITERAL+ SASL-IR LOGIN-REFERRALS ID ENABLE IDLE AUTH=PLAIN] Dovecot ready.
tag login dojomi@labs.local dojomi
tag LIST "" "*" 
* LIST (\HasNoChildren) "." INBOX
tag OK List completed.
tag LOGOUT
* BYE Logging out
tag OK Logout completed.
Connection closed by foreign host.
```

```shell
# postfix with tls
# /etc/postfix/main.cf
# TLS
postconf -e 'smtpd_use_tls = yes'
postconf -e 'smtpd_tls_security_level = may'
postconf -e 'smtpd_tls_auth_only = yes'
postconf -e 'smtpd_tls_key_file = /etc/pki/tls/certs/server.key'
postconf -e 'smtpd_tls_cert_file = /etc/pki/tls/certs/server.crt'
postconf -e 'smtpd_tls_loglevel = 1'
postconf -e 'smtpd_tls_received_header = yes'
postconf -e 'smtpd_tls_session_cache_timeout = 3600s'
postconf -e 'tls_random_source = dev:/dev/urandom'
# SASL
postconf -e 'smtpd_sasl_type = dovecot'
postconf -e 'broken_sasl_auth_clients = yes'
postconf -e 'smtpd_sasl_path = private/auth'
postconf -e 'smtpd_sasl_auth_enable = yes'
postconf -e 'smtpd_sasl_security_options = noanonymous'
postconf -e 'smtpd_recipient_restrictions = permit_sasl_authenticated, permit_mynetworks, reject_unauth_destination'
postconf -e 'smtpd_relay_restrictions = permit_sasl_authenticated, permit_mynetworks, reject_unauth_destination'
vim /etc/postfix/master.cf 
submission inet n       -       n       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
systemctl restart postfix
postfix check
netstat -ntlp
firewall-cmd --permanent --add-port=587/tcp
firewall-cmd --reload

telnet localhost 587
Trying ::1...
Connected to localhost.
Escape character is '^]'.
220 mail-server.labs.local ESMTP Postfix (2.10.1)
ehlo localhost
250-mail.labs.local.local
250-PIPELINING
250-SIZE 10485760
250-VRFY
250-ETRN
250-STARTTLS
250-ENHANCEDSTATUSCODES
250-8BITMIME
250 DSN
quit
```

```shell
# dovecot with ssl
# configure ssl-certificate
cd /etc/pki/tls/certs 
openssl genrsa -out server.key 1024
openssl req -new -key server.key -out server.csr
openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt
vim /etc/dovecot/conf.d/10-auth.conf
disable_plaintext_auth = yes
vim /etc/dovecot/conf.d/10-ssl.conf
ssl = yes
ssl_cert = </etc/pki/tls/certs/server.crt
ssl_key = </etc/pki/tls/certs/server.key
systemctl restart postfix
systemctl restart dovecot
systemctl restart saslauthd

openssl s_client -starttls smtp -connect localhost:587
250 DSN
ehlo localhost
250-mail.labs.local.local
250-PIPELINING
250-SIZE 10485760
250-VRFY
250-ETRN
250-AUTH PLAIN LOGIN
250-ENHANCEDSTATUSCODES
250-8BITMIME
250 DSN

doveadm auth test -a /var/spool/postfix/private/auth dojomi@labs.local
Password: 
passdb: dojomi@labs.local auth succeeded
    extra fields:
user=dojomi@labs.local

openssl s_client -crlf -connect localhost:995
user dojomi@lbx.io
+OK
pass dojomi
+OK Logged in.

netstat -ntlp
# 995 for POP3s, 587 for SMPT (SASL) and 993 for IMAPs
firewall-cmd --permanent --add-port={993/tcp,995/tcp,587/tcp}
firewall-cmd --permanent --remove-port={110/tcp,143/tcp}
firewall-cmd --reload
yum install -y epel-release 
yum install -y swaks
swaks --to dojomi@labs.local
# if u get error check entries on dns-server
# swaks --to dojomi@labs.local
# === Trying labs.local:25...
# *** Error connecting to labs.local:25:
# ***   IO::Socket::INET6: getaddrinfo: Name or service not known
```

```shell
# add a new user
echo"<name>@labs.local       labs.local/<name>/" >> /etc/postfix/vmailbox
doveadm pw -s SHA512-CRYPT >> /etc/dovecot/users
<name>@lab.local:{SHA}::::
```

![image](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/mail/1.png)

```shell
# squirrelmail
yum install -y squirrelmail
cd /usr/share/squirrelmail/config/
./conf.pl
# General
# -------
# 1.  Domain                 : labs.local
# 2.  Invert Time            : false
# 3.  Sendmail or SMTP       : SMTP
#
IMAP Settings
--------------
# 4.  IMAP Server            : 192.168.1.11
# 5.  IMAP Port              : 993
# 6.  Authentication type    : login
# 7.  Secure IMAP (TLS)      : true
# 8.  Server software        : dovecot
# 9.  Delimiter              : /
#
# SMTP Settings
#-------------
# 4.   SMTP Server           : 192.168.1.11
# 5.   SMTP Port             : 465
# 6.   POP before SMTP       : true
# 7.   SMTP Authentication   : login (with IMAP username and password) 
# 8.   Secure SMTP (TLS)     : true
# 9.   Header encryption key : 
#
# SMTP Settings
#-------------
# 1.   Default Folder Prefix : var/mail/vhosts
yum install -y httpd
firewall-cmd --permanent --add-service http
firewall-cmd --reload
systemctl enable httpd; systemctl start httpd
cat >> /etc/httpd/conf/httpd.conf <<eof
Alias /webmail /usr/share/squirrelmail
<Directory /usr/share/squirrelmail>
Options Indexes FollowSymLinks
RewriteEngine On
AllowOverride All
DirectoryIndex index.php
Order allow,deny
Allow from all
</Directory>
eof
systemctl restart httpd
firefox 192.168.1.11/webmail
# Error connecting to IMAP server: localhost.
# 13 : Permission denied
aureport -a
httpd system_u:system_r:httpd_t:s0 42 tcp_socket name_connect system_u:object_r:pop_port_t:s0 denied 1831
setsebool httpd_can_sendmail=1
firefox 192.168.1.11/webmail
# dojomi@labs.local
# dojomi
```

For configuration file
<https://github.com/DoJoMi/centbox_files/tree/master/mail>

## Amavisd/Spamassassin/Clamav for postfix

```shell
yum install -y epel-release
yum install -y amavisd-new spamassassin clamav-server clamav-server-systemd clamav-devel clamav-lib clamav-data clamav-update clamav-filesystem
cp -rv /etc/sysconfig/freshclam /etc/sysconfig/freshclam.bak
cp -rv /etc/amavisd/amavisd.conf /etc/amavisd/amavisd.conf.bak
vim /etc/amavisd/amavisd.conf
$mydomain = 'labs.local';
$myhostname = 'mail-server.labs.local';
cat >> /etc/postfix/master.cf <<eof
$ # Amavisd
amavisfeed unix - - n - 2 lmtp
        -o lmtp_data_done_timeout=1200
        -o lmtp_send_xforward_command=yes
127.0.0.1:10025 inet n - n - - smtpd
        -o content_filter=
        -o smtpd_delay_reject=no
        -o smtpd_client_restrictions=permit_mynetworks,reject
        -o smtpd_helo_restrictions=
        -o smtpd_sender_restrictions=
    -o smtpd_recipient_restrictions=permit_mynetworks,reject
        -o smtpd_data_restrictions=reject_unauth_pipelining
        -o smtpd_end_of_data_restrictions=
        -o smtpd_restriction_classes=
        -o mynetworks=127.0.0.0/8
        -o smtpd_error_sleep_time=0
        -o smtpd_soft_error_limit=1001
        -o smtpd_hard_error_limit=1000
        -o smtpd_client_connection_count_limit=0
        -o smtpd_client_connection_rate_limit=0
        -o receive_override_options=no_header_body_checks,no_unknown_recipient_checks,no_milters,no_address_mappings
        -o local_header_rewrite_clients=
        -o smtpd_milters=
        -o local_recipient_maps=
        -o relay_recipient_maps=
eof
cat >> /etc/postfix/main.cf <<eof
#use amavisd as filter on port 10024
content_filter = amavisfeed:[127.0.0.1]:10024
eof
systemctl enable spamassassin; systemctl start spamassassin
systemctl enable amavisd; systemctl start amavisd
systemctl restart postfix
sed -i -e 's/^Example/#Example/' /etc/freshclam.conf
freshclam
sa-update -D
# exit code is 0 --> systemctl restart spamassassin
systemctl restart postfix
systemctl restart amavisd
systemctl restart spamassassin
netstat -ntlp
telnet localhost 10024
Trying ::1...
Connected to localhost.
Escape character is '^]'.
220 [::1] ESMTP amavisd-new service ready
ehlo localhost
250-[::1]
250-VRFY
250-PIPELINING
250-SIZE
250-ENHANCEDSTATUSCODES
250-8BITMIME
250-SMTPUTF8
250-DSN
250 XFORWARD NAME ADDR PORT PROTO HELO IDENT SOURCE
quit
```

```shell
# testing virus detection
yum install -y swaks
# send mail with body --> file is stored inside /var/spool/amavisd/tmp/amavis change it under 
swaks -f dojomi@labs.local -t admin@labs.local --body "X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*"
tail -f /var/log/maillog
Jun  7 08:06:51 mail-server amavis[3343]: (03343-01) Blocked INFECTED (Eicar-Test-Signature) {DiscardedInternal,Quarantined}, MYNETS LOCAL [192.168.1.11]:51841 
```

```shell
# testing spam detection
# send mail with body --> file is stored inside /var/spool/amavisd/tmp/amavis
swaks -f dojomi@labs.local -t admin@labs.local --body "XJS*C4JDBQADN1.NSBN3*2IDNEN*GTUBE-STANDARD-ANTI-UBE-TEST-EMAIL*C.34X"
tail -f /var/log/maillog
Jun  7 08:11:11 mail-server amavis[3344]: (03344-02) Blocked SPAM {DiscardedInternal,Quarantined}, MYNETS LOCAL [192.168.1.11]:51853 <admin@labs.local> -> <admin@labs.local>, Queue-ID: 43AA022F83, Message-ID: <1f4d5e73a72c483506e30bb89486f95c.squirrel@192.168.1.11>, mail_id: l5qHxBfdPixC, Hits: 997.575, size: 885, 1666 ms
```
