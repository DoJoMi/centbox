# Logging

------------------------------------------------------------------------

## rsyslog

```shell
# server
nmcli general hostname cent01.labs.local
systemctl restart systemd-hostnamed
cat > /etc/hosts <<eof
192.168.1.11 cent02.labs.local cent02
eof
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

```shell
# client
nmcli general hostname cent02.labs.local
systemctl restart systemd-hostnamed
cat > /etc/hosts <<eof
192.168.1.10 cent01.labs.local cent02
eof
cp -pv /etc/rsyslog.conf /etc/rsyslog.conf.bak
echo "*.* @@192.168.1.10:514" >> /etc/rsyslog.conf
systemctl restart rsyslog.service
```

```shell
# server
ls -la /var/log/rsyslog/
# drwx------. 2 root root   78 May 10 05:42 cent01
# drwx------. 2 root root   61 May 10 05:45 cent02
```

## graylog

```shell
yum update -y
yum install -y java-1.8.0-openjdk-headless.x86_64 epel-release vim firewalld curl policycoreutils
systemctl enable firewalld && systemctl start firewalld
cat > /etc/yum.repos.d/mongodb-org.repo <<\eof
[mongodb]
name=MongoDB Repository
baseurl=http://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.2/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.2.asc
eof
yum install -y mongodb-org-4.2.2 mongodb-org-server-4.2.2 mongodb-org-shell-4.2.2 mongodb-org-mongos-4.2.2 mongodb-org-tools-4.2.2
systemctl daemon-reload && systemctl enable mongod.service && systemctl start mongod.service
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
cat > /etc/yum.repos.d/elasticsearch.repo <<eof
[elasticsearch-7.x]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/oss-7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch    
enabled=1
autorefresh=1
type=rpm-md
eof
yum install -y elasticsearch-oss
echo "action.auto_create_index: false" >> /etc/elasticsearch/elasticsearch.yml
systemctl daemon-reload && systemctl enable elasticsearch.service
systemctl start elasticsearch.service
rpm -Uvh https://packages.graylog2.org/repo/packages/graylog-3.1-repository_latest.rpm
yum install -y graylog-server
cp /etc/graylog/server/server.conf /etc/graylog/server/server.conf.bak
yum install -y pwgen
secret=$(pwgen -N 1 -s 96)
sed -i "s/^password_secret =/password_secret = $secret/g" /etc/graylog/server/server.conf
root_password_sha2=$(echo -n && head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1)
# graylog
# 4bbdd5a829dba09d7a7ff4c1367be7d36a017b4267d728d31bd264f63debeaa6
sed -i "s/^root_password_sha2 =/root_password_sha2 = $root_password_sha2/g" /etc/graylog/server/server.conf
ip=$(ip a | grep inet | tail -3 | head -1 | awk '{print $2}' | awk -F / '{print $1}')
echo "http_bind_address = $ip:9000" >> /etc/graylog/server/server.conf
sed -i "s/^elasticsearch_shards = 4/elasticsearch_shards = 1/g " /etc/graylog/server/server.conf
systemctl daemon-reload && systemctl enable graylog-server.service
systemctl start graylog-server
firewall-cmd --zone=public --add-port=9000/tcp --permanent
firewall-cmd --reload
# Allow the web server to access the network:
setsebool -P httpd_can_network_connect 1
# Graylog REST API and web interface:
semanage port -a -t http_port_t -p tcp 9000
# Elasticsearch (only if the HTTP API is being used):
semanage port -a -t http_port_t -p tcp 9200
# Allow using MongoDB default port (27017/tcp):
semanage port -a -t mongod_port_t -p tcp 27017
```
