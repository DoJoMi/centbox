# Monitoring

------------------------------------------------------------------------

## Cockpit

```shell
yum install -y cockpit httpd
systemctl enable cockpit.socket; systemctl start cockpit.service
systemctl enable httpd.service; systemctl start httpd.service
firewall-cmd --permanent --add-service={cockpit,http}
firewall-cmd --reload
vim /usr/lib/systemd/system/cockpit.service
ExecStart=/usr/libexec/cockpit-ws --no-tls 
systemctl daemon-reload; systemctl restart cockpit; systemctl restart httpd
firefox 192.168.1.11:9090
# user:your_root
```

## Nagios

```shell
yum install -y httpd httpd-tools php gcc glibc glibc-common gd gd-devel make net-snmp vim wget ntp unzip -y
ntpdate pool.ntp.org
useradd nagios && groupadd nagcmd
usermod -G nagcmd nagios && usermod -G nagcmd apache
mkdir /root/nagios
cd /root/nagios
wget https://assets.nagios.com/downloads/nagioscore/releases/nagios-4.4.2.tar.gz
wget https://nagios-plugins.org/download/nagios-plugins-2.2.1.tar.gz
tar -xvf nagios-4.4.2.tar.gz
tar -xvf nagios-plugins-2.2.1.tar.gz
cd nagios-4.4.2/
./configure --with-command-group=nagcmd
make all; make install; make install-init; make install-commandmode;make install-config
sed -i 's/nagios@localhost/youremail@yourdomain/g' /usr/local/nagios/etc/objects/contacts.cfg
make install-webconf
htpasswd -s -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
systemctl enable httpd && systemctl start httpd
# nagios plugins
cd /root/nagios
cd nagios-plugins-2.2.1/
./configure --with-nagios-user=nagios --with-nagios-group=nagios
make; make install
/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
systemctl enable nagios && systemctl start nagios
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
rm -rf /root/nagios
```

## Icinga

```shell
# server
echo "192.168.1.20 icinga.labs.local imaster" >> /etc/hosts
yum install -y wget httpd mod_ssl gd gd-devel mariadb-server php-mysql php-xmlrpc gcc mariadb libdbi libdbi-devel libdbi-drivers libdbi-dbd-mysql
useradd icinga
groupadd icinga-cmd
usermod -aG icinga-cmd icinga
usermod -aG icinga-cmd apache
cd /tmp/
wget http://downloads.sourceforge.net/project/icinga/icinga/1.10.1/icinga-1.10.1.tar.gz
tar -xvzf icinga-1.10.1.tar.gz 
cd icinga-1.10.1
./configure --with-command-group=icinga-cmd --enable-idoutils
make all
make install
make install-init
make install-config
make install-config
make install-commandmode
make install-webconf
make install-idoutils
vim /usr/local/icinga/etc/objects/contacts.cfg
cd /usr/local/icinga/etc/
mv idomod.cfg-sample idomod.cfg
mv ido2db.cfg-sample ido2db.cfg
cd modules/
mv idoutils.cfg-sample idoutils.cfg
sytemctl start mariadb.service
systemctl start mariadb.service
mysql -u root -p
mysql -u root -p icinga < /tmp/icinga-1.10.1/module/idoutils/db/mysql/mysql.sql 
htpasswd -c /usr/local/icinga/etc/htpasswd.user icingaadmin
systemctl restart httpd.service
cd /tmp/
wget http://nagios-plugins.org/download/nagios-plugins-2.0.3.tar.gz
tar xvzf /tmp/nagios-plugins-2.0.3.tar.gz 
cd /tmp/nagios-plugins-2.0.3
configure --prefix=/usr/local/icinga --with-cgiurl=/icinga/cgi-bin --with-nagios-user=icinga --with-nagios-group=icinga
./configure --prefix=/usr/local/icinga --with-cgiurl=/icinga/cgi-bin --with-nagios-user=icinga --with-nagios-group=icinga
make
make install
/usr/local/icinga/bin/icinga -v /usr/local/icinga/etc/icinga.cfg
systemctl start icinga
/etc/init.d/icinga start
/etc/init.d/ido2db start
chkconfig ido2db on
chkconfig icinga on
systemctl enable httpd
systemctl enable mariadb
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
netstat -ntlp
htpasswd -c /usr/local/icinga/etc/htpasswd.users icingaadmin
# echo "Home Page" > /var/www/html/index.html
firefox 192.168.1.20/icinga/
```

## Icinga2

```shell
echo "192.168.1.20 ic1.labs.local imaster" >> /etc/hosts
rpm --import http://packages.icinga.org/icinga.key
rpm -i http://packages.icinga.org/epel/7/release/noarch/icinga-rpm-release-7-1.el7.centos.noarch.rpm 
yum makecache
yum install -y icinga2 wget icinga2-ido-mysql
yum -y install mariadb-server mariadb
systemctl enable mariadb
systemctl start mariadb
mysql_secure_installation
mysql -u root -p
create database icinga2;
grant all on icinga2.* to 'icinga2'@'localhost' identified by 'icinga-password';
flush privileges;
quit
mysql -u root -p icinga2 < /usr/share/icinga2-ido-mysql/schema/mysql.sql

cat > /etc/icinga2/features-available/ido-mysql.conf <<eof
library "db_ido_mysql"  
object IdoMysqlConnection "ido-mysql" {
user = "icinga2"
password = "icinga-password"
host = "localhost"
database="icinga2"                                                                                                
}
eof
icinga2 feature enable ido-mysql
systemctl restart icinga2
```

```shell
# install icinga2-web-interface
yum install -y httpd php-gd php-intl php-ZendFramework php-pear php-pdo php-soap php-ldap php-cli php-common php-devel php-mbstring php-mysql php-xml
usermod -aG icingacmd apache
yum install -y epel-release
yum install -y php-pecl-imagick 
yum install -y icingaweb2 icingacli nagios-plugins-all
systemctl enable httpd
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
systemctl restart icinga2 
systemctl start httpd

# from cmd
icingacli setup token create
# The newly generated setup token is: 0f785cff525840f1
firefox http://192.168.1.20/icingaweb2/setup

# The PHP config 'date.timezone' is not defined.
vim /etc/php.ini
date.timezone = "Europe/Vienna"
systemctl restart httpd 

# The directory /etc/icingaweb2 is not writable.
yum install -y policycoreutils-python
icingacli setup config directory
ls -laZ /etc/icingaweb2/
# drwxrws---. root icingaweb2 system_u:object_r:httpd_sys_rw_content_t:s0 .
semanage fcontext -a -t httpd_sys_rw_content_t '/etc/icingaweb2(/.*)?'
restorecon -Rv "/etc/icingaweb2"
# refresh webpage
```

![image](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/monitoring/1.png)

![image](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/monitoring/2.png)

![image](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/monitoring/3.png)

![image](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/monitoring/4.png)

![image](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/monitoring/5.png)

![image](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/monitoring/6.png)

![image](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/monitoring/7.png)

![image](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/monitoring/8.png)

![image](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/monitoring/9.png)

![image](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/monitoring/10.png)

![image](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/monitoring/11.png)


Icinga2 - Remote Hosts

```shell
# master
echo "192.168.1.20 ic1.labs.local imaster" >> /etc/hosts
echo "192.168.1.21 ic2.labs.local inode" >> /etc/hosts
icinga2 node wizard
#Please specify if this is a satellite setup ('n' installs a master setup) [Y/n]: n
#Starting the Master setup routine...
#Please specifiy the common name (CN) [ic1.labs.local]: 
#Checking for existing certificates for common name 'ic1.labs.local'...
#Certificates not yet generated. Running 'api setup' now.
#information/cli: Generating new CA.
#information/base: Writing private key to '/var/lib/icinga2/ca/ca.key'.
#information/base: Writing X509 certificate to '/var/lib/icinga2/ca/ca.crt'.
#information/cli: Generating new CSR in '/etc/icinga2/pki/ic1.labs.local.csr'.
#information/base: Writing private key to '/etc/icinga2/pki/ic1.labs.local.key'.
#information/base: Writing certificate signing request to '/etc/icinga2/pki/ic1.labs.local.csr'.
#information/cli: Signing CSR with CA and writing certificate to '/etc/icinga2/pki/ic1.labs.local.crt'.
#information/cli: Copying CA certificate to '/etc/icinga2/pki/ca.crt'.
#Generating master configuration for Icinga 2.
#information/cli: Adding new ApiUser 'root' in '/etc/icinga2/conf.d/api-users.conf'.
#information/cli: Enabling the 'api' feature.
#Enabling feature api. Make sure to restart Icinga 2 for these changes to take effect.
#information/cli: Dumping config items to file '/etc/icinga2/zones.conf'.
#information/cli: Created backup file '/etc/icinga2/zones.conf.orig'.
#Please specify the API bind host/port (optional):
#Bind Host []: 
#Bind Port []: 
#information/cli: Created backup file '/etc/icinga2/features-available/api.conf.orig'.
#information/cli: Updating constants.conf.
#information/cli: Created backup file '/etc/icinga2/constants.conf.orig'.
#information/cli: Updating constants file '/etc/icinga2/constants.conf'.
#information/cli: Updating constants file '/etc/icinga2/constants.conf'.
#information/cli: Updating constants file '/etc/icinga2/constants.conf'.
#Done.
systemctl restart icinga2
firewall-cmd --permanent --add-port={5665/tcp,5665/udp}
firewall-cmd --reload
```

```shell
# client
echo "192.168.1.20 ic1.labs.local imaster" >> /etc/hosts
echo "192.168.1.21 ic2.labs.local inode" >> /etc/hosts
yum install -y epel-release
rpm --import http://packages.icinga.org/icinga.key
rpm -i http://packages.icinga.org/epel/7/release/noarch/icinga-rpm-release-7-1.el7.centos.noarch.rpm
yum makecache
yum install -y icinga2 nagios-plugin-all
firewall-cmd --permanent --add-port={5665/tcp,5665/udp}
firewall-cmd --reload
icinga2 node wizard
#Please specify if this is a satellite setup ('n' installs a master setup) [Y/n]: 
#Starting the Node setup routine...
#Please specifiy the common name (CN) [ic2.labs.local]: 
#Please specify the master endpoint(s) this node should connect to:
#Master Common Name (CN from your master setup): ic1.labs.local
#Do you want to establish a connection to the master from this node? [Y/n]: 
#Please fill out the master connection information:
#Master endpoint host (Your master's IP address or FQDN): 192.168.1.11
#Master endpoint port [5665]: 
#Add more master endpoints? [y/N]: 
#Please specify the master connection for CSR auto-signing (defaults to master endpoint host):
#Host [192.168.1.11]: 
#Port [5665]: 
#information/base: Writing private key to '/etc/icinga2/pki/ic2.labs.local.key'.
#information/base: Writing X509 certificate to '/etc/icinga2/pki/ic2.labs.local.crt'.
#information/cli: Fetching public certificate from master (192.168.1.11, 5665):
#Certificate information:
# Subject:     CN = ic1.labs.local
# Issuer:      CN = Icinga CA
# Valid From:  Jun  8 11:36:52 2016 GMT
# Valid Until: Jun  5 11:36:52 2031 GMT
# Fingerprint: 76 5C 1E A5 46 1A A5 83 E6 55 E6 C0 53 33 1F B3 9E 03 4A XX 
#Is this information correct? [y/N]: y
#information/cli: Received trusted master certificate.
#Please specify the request ticket generated on your Icinga 2 master.
# (Hint: # icinga2 pki ticket --cn 'ic2.labs.local'): 4624da4df0a91f79ebeedff444ff7f6dea084710
#information/cli: Requesting certificate with ticket '4624da4df0a91f79ebeedff444ff7f6dea084710'.
#information/cli: Created backup file '/etc/icinga2/pki/ic2.labs.local.crt.orig'.
#information/cli: Writing signed certificate to file '/etc/ic22/pki/ic2.labs.local.crt'.
#information/cli: Writing CA certificate to file '/etc/ic2/pki/ca.crt'.
#Please specify the API bind host/port (optional):
#Bind Host []: 
#Bind Port []: 
#Accept config from master? [y/N]: y
#Accept commands from master? [y/N]: y
#information/cli: Disabling the Notification feature.
#Disabling feature notification. Make sure to restart Icinga 2 for these changes to take effect.
#information/cli: Enabling the Apilistener feature.
#Enabling feature api. Make sure to restart Icinga 2 for these changes to take effect.
#information/cli: Created backup file '/etc/icinga2/features-available/api.conf.orig'.
#information/cli: Generating local zones.conf.
#information/cli: Dumping config items to file '/etc/icinga2/zones.conf'.
#information/cli: Created backup file '/etc/icinga2/zones.conf.orig'.
#information/cli: Updating constants.conf.
#information/cli: Created backup file '/etc/icinga2/constants.conf.orig'.
#information/cli: Updating constants file '/etc/icinga2/constants.conf'.
#information/cli: Updating constants file '/etc/icinga2/constants.conf'.
#Done.
systemctl restart icinga2
```

```shell
# master
icinga2 node list
icinga2 node update-config
systemctl restart icinga2
```
