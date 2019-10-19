# FTP

------------------------------------------------------------------------

## VSFTP

```shell
# local user
yum install -y vsftpd
# where are the files located
rpm -qc vsftpd 
firewall-cmd --permanent --add-service=ftp
firewall-cmd --reload
systemctl enable vsftpd
systemctl start vsftpd
```

```shell
# logging
vim /etc/vsftpd/vsftpd.conf
# xferlog_enable=YES
# xferlog_file=/var/log/vsftpd.log 
# xferlog_std_format=NO
# log_ftp_protocol=YES
# debug_ssl=YES
systemctl restart vsftpd
# do not create /var/log/vsftpd.log file it will be created after first login
ftp root@192.168.1.11
# 530 Permission denied.
chmod 600 /var/log/vsftpd.log
# if u want to check packets are receive
sudo yum install -y tcpdump
tcpdump 'src 192.168.1.2 and dst 192.168.1.11 and port ftp'
```

```shell
# testing
# allow connect user root
vim /etc/vsftpd/ftpusers
##root
# vim /etc/vsftpd/user_list
#root
# or create a real user
useradd -m -d /home/dojomi -s /bin/bash -c "dojomi" -u 1001 -g ftp dojomi
passwd dojomi
touch /home/dojomi/file
# on client side
ftp dojomi@192.168.1.11
# pwd>
# ls
# get file
# 550 Failed to open file.
# on server side
tail -f /var/log/vsftpd.log
aureport -a || ausearch -m avc
unconfigured_u:object_r:user_home_t:s0 denied
getsebool -a | grep ftp
setsebool -P ftp_home_dir 1
# on client side
# ftp> get file
# ftp> 226 Transfer complete
```

```shell
# chroot user
vim /etc/vsftpd/vsftpd.conf
# all users are jailed by default:
# chroot_local_user=YES
# chroot_list_enable=NO
# some users are jailed inside /etc/vsftpd.chroot_list --> default config for me
# chroot_local_user=NO
# chroot_list_enable=YES
# some users inside /etc/vsftpd/chroot_list declared as free users
# chroot_local_user=YES
# chroot_list_enable=YES
allow_writeable_chroot=YES
chroot_list_file=/etc/vsftpd/chroot_list

echo -e "dojomi" >> /etc/vsftpd/chroot_list
systemctl restart vsftpd
touch /home/dojomi/file
# on client side
ftp dojomi@192.168.1.11
# Connected to 192.168.1.11.
# 220 (vsFTPd 3.0.2)
# 331 Please specify the password.
# Password: 
# 230 Login successful.
# Remote system type is UNIX.
# Using binary mode to transfer files.
# ftp> pwd
# Remote directory: /
# ftp> cd /home
# 550 Failed to change directory.
# ftp> ls
# 229 Entering Extended Passive Mode (|||41582|).
# 150 Here comes the directory listing.
# -rw-r--r--    1 ftp      ftp             0 Apr 20 11:22 file
# 226 Directory send OK.
```

```shell
# specific users configurations
vim /etc/vsftpd/vsftpd.conf
# write_enable=YES
user_config_dir=/etc/vsftpd/vsftpd_user_conf/$USER

mkdir -p /etc/vsftpd/vsftpd_user_conf/
cat > /etc/vsftpd/vsftpd_user_conf/dojomi <<eof
dirlist_enable=YES
download_enable=YES
local_root=/ftp/virtual/dojomi
write_enable=NO
eof
```

```shell
# some other usefull modifications
vim /etc/vsftpd/vsftpd.conf
anonymous_enable=NO
anon_uplad_enable=NO
anon_mkdir_write_enable=NO
ascii_upload_enable=YES
ascii_download_enable=YES
ftpd_banner= FTP-Server
ls_recurse_enable=YES
dual_log_enable=YES
use_localtime=YES
local_max_rate=1000000 
max_clients=50         
max_per_ip=2 
```

```shell
# ssl
mkdir /etc/ssl/private
openssl req -x509 -nodes -days 365 -newkey rsa:1024 -keyout /etc/ssl/private/vsftpd.pem -out /etc/ssl/private/vsftpd.pem
vim /etc/vsftpd/vsftpd.conf
ssl_enable=YES
allow_anon_ssl=NO
force_local_data_ssl=NO
force_local_logins_ssl=NO
ssl_tlsv1=YES
ssl_sslv2=NO
ssl_sslv3=NO
require_ssl_reuse=NO
ssl_ciphers=HIGH
rsa_cert_file=/etc/ssl/private/vsftpd.pem
rsa_private_key_file=/etc/ssl/private/vsftpd.pem

vim /etc/vsftpd/vsftpd.conf
pam_service_name=vsftpd

systemctl restart vsftpd
# on client side
sftp://root@192.168.1.11
```

```shell
# virtual users with pam\_pwfile.so
yum install -y httpd-tools
# delete real user
userdel -r dojomi
mkdir -p /ftp/virtual/dojomi
useradd -m -d /ftp/virtual/dojomi -s /bin/false -c "dojomi" -u 1001 -n dojomi
# register a virtual user
htpasswd -c /etc/vsftpd/passwd dojomi
chmod 600 /etc/vsftpd/passwd
cat /etc/vsftpd/passwd
vim /etc/vsftpd/vsftpd.conf
pam_service_name=vsftpd_virtual
user_sub_token=$USER
local_root=/ftp/virtual/$USER
virtual_use_local_privs=YES
userlist_enable=YES
userlist_deny=YES
userlist_file=/etc/vsftpd/ftpusers
hide_ids=YES
tcp_wrappers=YES

echo -e "dojomi" > /etc/vsftpd/vsftpd_userlist
vim /etc/pam.d/vsftpd_virtual
auth required pam_pwdfile.so pwdfile /etc/vsftpd/passwd
account required pam_permit.so

systemctl restart vsftpd
tail -f /var/log/secure
# localhost vsftpd[13315]: PAM adding faulty module: /usr/lib64/security/pam_pwdfile.so
sudo yum install -y pam-devel
cd /tmp
curl -O http://springdale.math.ias.edu/data/puias/unsupported/6/x86_64/pam_pwdfile-0.99-1.puias6.x86_64.rpm
rpm -Uvh pam_pwdfile-0.99-1.puias6.x86_64.rpm
ls -l /lib64/security/pam_p*
htpasswd -d /etc/vsftpd/passwd dojomi
# on client side    
ftp dojomi@192.168.1.11
# Connected to 192.168.1.11.
# 220 (vsFTPd 3.0.2)
# 331 Please specify the password.
# Password: 
# 230 Login successful.
# ls
# 229 Entering Extended Passive Mode (|||52645|).
# 150 Here comes the directory listing.
# 226 Directory send OK.
aureport -a
vsftpd system_u:system_r:ftpd_t:s0-s0:c0.c1023 257 dir read unconfined_u:object_r:default_t:s0 denied 432
yum provides */semanage
yum install policycoreutils-python
semanage fcontext -a -t public_content_rw_t '/ftp/virtual(/.*)?'
restorecon -R /ftp/virtual/*
# ls
# 229 Entering Extended Passive Mode (|||27285|).
# 150 Here comes the directory listing.
# -rw-r--r--    1 ftp      ftp             0 Apr 20 11:04 new
# 226 Directory send OK.
```

```shell
# virtual users with db4-utils
yum install epel-release -y
yum install db4-utils db4 -y
# add username and pwd one by one 
vim /home/virtual_users.txt
dojomi
centos
db_load -T -t hash -f /home/virtual_users.txt /etc/vsftpd/virtual_users.db
# create a new auth file
vim /etc/pam.d/vsftpd_db4
auth    required        pam_userdb.so   db=/etc/vsftpd/virtual_users
account required        pam_userdb.so   db=/etc/vsftpd/virtual_users
session required        pam_loginuid.so

vim /etc/vsftpd/vsftpd.conf
pam_service_name=vsftpd_db4
# all other options from the default file

systemctl restart vsftpd
tail -f /var/log/secure
# on client side
ftp dojomi@192.168.1.11
# Connected to 192.168.1.11.
# 220 (vsFTPd 3.0.2)
# 331 Please specify the password.
# Password: centos
# 230 Login successful.
```

```shell
# virtual users with mysql
yum install -y mariadb-server mariadb wget
systemctl enable mariadb.service
systemctl start mariadb.service
mysql_secure_installation
# pam_mysql is not in standard repos included
cd /tmp
wget ftp://ftp.pbone.net/mirror/archive.fedoraproject.org/fedora/linux/releases/20/Everything/x86_64/os/Packages/p/pam_mysql-0.7-0.16.rc1.fc20.x86_64.rpm
rpm -Uvh pam_mysql-0.7-0.16.rc1.fc20.x86_64.rpm
mysql -u root -p
# MariaDB [(none)]> CREATE DATABASE vsftpd;
# MariaDB [(none)]> GRANT SELECT ON vsftpd.* TO 'vsftpd'@'localhost' IDENTIFIED BY 'vsftpd'; grant all privileges on vsftpd.* TO 'vsftpd'@'localhost' IDENTIFIED BY '';
# MariaDB [(none)]> FLUSH PRIVILEGES;
# MariaDB [(none)]> USE vsftpd;
# MariaDB [vsftpd]> CREATE TABLE `accounts`(`id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,`username` VARCHAR(40) NOT NULL,`passwd` VARCHAR(50) NOT NULL,UNIQUE(`username`)) ENGINE = INNODB;
# MariaDB [vsftpd]> insert into accounts(username, passwd) values ('dojomi', md5('dojomi'));
# MariaDB [vsftpd]> select * from accounts;
# MariaDB [vsftpd]> exit;
useradd -n -s /sbin/nologin vsftpd
# chmod 770 /ftp/virtual/dojomi/
# chown vsftpd. /ftp/virtual/dojomi/
vim /etc/vsftpd/vsftpd.conf
pam_service_name=vsftpd_mariadb
vim /etc/pam.d/vsftpd_mariadb
session     optional     pam_keyinit.so     force revoke
auth required pam_mysql.so user=vsftpd passwd=vsftpd host=localhost db=vsftpd table=accounts usercolumn=username passwdcolumn=passwd crypt=3
account required pam_mysql.so user=vsftpd passwd=vsftpd host=localhost db=vsftpd table=accounts usercolumn=username passwdcolumn=passwd crypt=3

systemctl restart vsftpd
tail -f /var/log/secure
# on client side
ftp dojomi@192.168.1.11
# Connected to 192.168.1.11.
# 220 (vsFTPd 3.0.2)
# 331 Please specify the password.
# Password: 
# 230 Login successful.
```

For configuration files
<https://github.com/DoJoMi/centbox_files/tree/master/vsftpd>
