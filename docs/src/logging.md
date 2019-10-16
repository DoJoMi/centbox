# Logging

------------------------------------------------------------------------

## rsyslog

server

```shell
nmcli general hostname cent01.labs.local
systemctl restart systemd-hostnamed
cat > /etc/hosts <<EOF
192.168.1.11 cent02.labs.local cent02
EOF
cp -pv /etc/rsyslog.conf /etc/rsyslog.conf.bak
vim /etc/rsyslog.conf
# uncommend
$ModLoad imklog
$ModLoad imtcp
$InputTCPServerRun 514
# @end of file
$template TmplAuth, "/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log" 
$template TmplMsg, "/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log" 
authpriv.*   ?TmplAuth
*.info,mail.none,authpriv.none,cron.none   ?TmplMsg

mkdir -p /var/log/rsyslog
yum install -y policycoreutils-python
semanage fcontext -a -t var_log_t "/var/log/rsyslog(/.*)?"
restorecon -R -v /var/log/rsyslog
firewall-cmd --permanent --add-port=514/tcp
firewall-cmd --reload
# or allow a specific ip
# firewall-cmd --permanent --zone=trusted--add-source=192.168.1.10/24
systemctl restart rsyslog.service
```

client

```shell
nmcli general hostname cent02.labs.local
systemctl restart systemd-hostnamed
cat > /etc/hosts <<EOF
192.168.1.10 cent01.labs.local cent02
EOF
cp -pv /etc/rsyslog.conf /etc/rsyslog.conf.bak
echo "*.* @@192.168.1.10:514" >> /etc/rsyslog.conf
systemctl restart rsyslog.service
```

server

```shell
ls -la /var/log/rsyslog/
# drwx------. 2 root root   78 May 10 05:42 cent01
# drwx------. 2 root root   61 May 10 05:45 cent02
```
