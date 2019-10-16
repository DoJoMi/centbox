# Syncronization

------------------------------------------------------------------------

## NTP

```shell
yum install -y ntp
cp /etc/ntp.conf /etc/ntp.conf.bak
cat > /etc/ntp.conf <<EOF
restrict default kod nomodify notrap nopeer noquery
restrict -6 default kod nomodify notrap nopeer noquery
restrict 127.0.0.1
restrict -6 ::1  
server 0.at.pool.ntp.org iburst
server 1.at.pool.ntp.org iburst
server 2.at.pool.ntp.org iburst
server 3.at.pool.ntp.org iburst
driftfile /var/lib/ntp/ntp.drift
logfile /var/log/ntp.log
EOF

# add firewall rule for clients
firewall-cmd --permanent --add-service=ntp
firewall-cmd --reload
systemctl enable ntpd
systemctl start ntpd
ntpq -p

# sync with Windows
# Taskbar
# Change Date and Time Settings
# Internet Time tab
# Change Settings
# Check Synchronize with an Internet time server
# put your server’s IP or FQDN on Server filed 
# Update now and get sync from server
```

## Chrony

```shell
yum install -y chrony
cp /etc/crony.conf /etc/crony.conf.bak
cat > /etc/crony.conf <<EOF
# http://www.pool.ntp.org/zone/at
server 0.at.pool.ntp.org
server 1.at.pool.ntp.org
server 2.at.pool.ntp.org
server 3.at.pool.ntp.org
allow 192.168/24
driftfile /etc/chrony.drift
rtconutc
rtcsync
EOF

systemctl start chronyd
systemctl enable chronyd
chronyc sources
chronyc activity
```
