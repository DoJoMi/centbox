## Database

------------------------------------------------------------------------

# Mariadb

```shell
nmcli general hostname db.labs.local
systemctl restart systemd-hostnamed
yum install -y mariadb mariadb-server mariadb-test
systemctl enable mariadb; systemctl start mariadb
netstat -ntlp
firewall-cmd --permanent --add-service=mysql
firewall-cmd --reload
systemctl restart mariadb
mysql_secure_installation
```

sample database

```shell
mysql -u root -p
create database music;
use music;
create table hiphop(singer varchar(40),title varchar(40),yt varchar(100), registration INT);
insert into hiphop(registration,singer,title,yt)values(1,'DMX','Ruff Ryders Anthem', 'https://youtu.be/ThlhSnRk21E');
insert into hiphop(registration,singer,title,yt)values(2,'Swollen Members','Full Contact','https://youtu.be/iVZ1pvJnmsM');
insert into hiphop(registration,singer,title,yt)values(3,'Dead Prez','Hip Hop','https://youtu.be/4jNyr6BJZuI');
insert into hiphop(registration,singer,title,yt)values(4,'The Fugees','Ready Or Not','https://youtu.be/aIXyKmElvv8');
select * from hiphop;
select * from hiphop where singer='';
```

create a user who can edit database

```shell
create user dojomi@'%' identified by 'secret';
grant select,insert,update,delete on music.*to dojomi@'%';
flush privileges;
quit;
mysql -u dojomi -p
system clear;
show databases;
```

change root pwd

```shell
mysql -u root -p 'root'
update mysql.user set password = password('new_pwd') where user = 'root';
flush privileges;
```

user login file

```shell
cat > .my.cnf <<EOF
user=dojomi
password=secret
EOF
```

### Master-Master Replication

dbmaster1

```shell
nmcli general hostname dbmaster1.labs.local
systemctl restart systemd-hostnamed
echo "192.168.1.13 dbmaster1.labs.local dbmaster1" >> /etc/hosts
echo "192.168.1.14 dbmaster2.labs.local dbmaster2" >> /etc/hosts
yum install -y mariadb mariadb-server
systemctl enable mariadb; systemctl start mariadb
netstat -ntlp
firewall-cmd --permanent --add-service=mysql
firewall-cmd --reload
systemctl restart mariadb
mysql_secure_installation
vim /etc/my.cnf
[mysqld]
server-id=1
log-bin=mysql-bin
mysql -u root -p
grant replication slave on *.* to repl@'%' identified by 'secret';
flush privileges;
flush table with read lock;
exit;
systemctl restart mariadb
mysql -u root -p
show master status;
# +------------------+----------+--------------+------------------+
# | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
# +------------------+----------+--------------+------------------+
# | mysql-bin.000001 |      245 |              |                  |
# +------------------+----------+--------------+------------------+
```

dbmaster2

```shell
nmcli general hostname dbmaster1.labs.local
systemctl restart systemd-hostnamed
echo "192.168.1.13 dbmaster1.labs.local dbmaster1" >> /etc/hosts
echo "192.168.1.14 dbmaster2.labs.local dbmaster2" >> /etc/hosts
yum install -y mariadb mariadb-server
systemctl enable mariadb; systemctl start mariadb
netstat -ntlp
firewall-cmd --permanent --add-service=mysql
firewall-cmd --reload
systemctl restart mariadb
mysql_secure_installation
vim /etc/my.cnf
[mysqld]
server-id=2
log-bin=mysql-bin
mysql -u root -p
grant replication slave on *.* to repl@'%' identified by 'secret';
flush privileges;
flush table with read lock;
mysql -u root -p
change master to
master_host='dbmaster1',
master_user='repl',                 #configured on dbmaster side
master_password='secret',           #configured on dbmaster side
master_log_file='mysql-bin.000001', #show master status; on dbmaster
master_log_pos=245;                 #show master status; on dbmaster
start slave;
show slave status\G;
#          Slave_IO_State: Waiting for master to send event
#          Master_Host: dbmaster1
#          Master_User: repl
#          Master_Port: 3306
#          Connect_Retry: 60
#          Master_Log_File: mysql-bin.000001
#          Read_Master_Log_Pos: 245
show master status;
# +------------------+----------+--------------+------------------+
# | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
# +------------------+----------+--------------+------------------+
# | mysql-bin.000001 |      245 |              |                  |
# +------------------+----------+--------------+------------------+
```

dbmaster1

```shell
mysql -u root -p
# change master to
master_host='dbmaster2',
master_user='repl',                 #configured on dbmaster side
master_password='secret',           #configured on dbmaster side
master_log_file='mysql-bin.000001', #show master status; on dbmaster
master_log_pos=245;                 #show master status; on dbmaster
show slave status\G;
# Slave_IO_State: Waiting for master to send event
#          Master_Host: dbmaster2
#          Master_User: repl
#          Master_Port: 3306
#          Connect_Retry: 60
#          Master_Log_File: mysql-bin.000001
#          Read_Master_Log_Pos: 245
create database music;
use music;
create table hiphop(singer varchar(40),title varchar(40),yt varchar(100), registration INT);
insert into hiphop(registration,singer,title,yt)values(1,'DMX','Ruff Ryders Anthem', 'https://youtu.be/ThlhSnRk21E');
select * from music.hiphop;
# +--------+--------------------+------------------------------+--------------+
# | singer | title              | yt                           | registration |
# +--------+--------------------+------------------------------+--------------+
# | DMX    | Ruff Ryders Anthem | https://youtu.be/ThlhSnRk21E |            1 |
# +--------+--------------------+------------------------------+--------------+
```

dbmaster2

```shell
select * from music.hiphop;
# +--------+--------------------+------------------------------+--------------+
# | singer | title              | yt                           | registration |
# +--------+--------------------+------------------------------+--------------+
# | DMX    | Ruff Ryders Anthem | https://youtu.be/ThlhSnRk21E |            1 |
# +--------+--------------------+------------------------------+--------------+
```

### Master-Slave Replication

dbmaster

```shell
nmcli general hostname dbmaster.labs.local
systemctl restart systemd-hostnamed
echo "192.168.1.13 dbmaster.labs.local dbmaster" >> /etc/hosts
echo "192.168.1.14 dbslave.labs.local dbslave" >> /etc/hosts
yum install -y mariadb mariadb-server
systemctl enable mariadb; systemctl start mariadb
netstat -ntlp
firewall-cmd --permanent --add-service=mysql
firewall-cmd --reload
systemctl restart mariadb
mysql_secure_installation
mysql -u root -p
create database music;
use music;
create table hiphop(singer varchar(40),title varchar(40),yt varchar(100), registration INT);
insert into hiphop(registration,singer,title,yt)values(1,'DMX','Ruff Ryders Anthem', 'https://youtu.be/ThlhSnRk21E');
insert into hiphop(registration,singer,title,yt)values(2,'Swollen Members','Full Contact','https://youtu.be/iVZ1pvJnmsM');
insert into hiphop(registration,singer,title,yt)values(3,'Dead Prez','Hip Hop','https://youtu.be/4jNyr6BJZuI');
exit;
vim /etc/my.cnf
[mysqld]
server-id=1
log-bin=mysql-bin
mysql -u root -p
grant replication slave on *.* to repl@'%' identified by 'secret';
flush privileges;
flush table with read lock;
exit;
systemctl restart mariadb
mysql -u root -p
show master status;
# +------------------+----------+--------------+------------------+
# | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
# +------------------+----------+--------------+------------------+
# | mysql-bin.000001 |      245 |              |                  |
# +------------------+----------+--------------+------------------+
mysqldump music > music.sql
scp music.sql root@dbslave:/root
```

dbslave

```shell
nmcli general hostname dbmaster.labs.local
systemctl restart systemd-hostnamed
echo "192.168.1.13 dbmaster.labs.local dbmaster" >> /etc/hosts
echo "192.168.1.14 dbslave.labs.local dbslave" >> /etc/hosts
yum install -y mariadb mariadb-server 
systemctl enable mariadb; systemctl start mariadb
firewall-cmd --permanent --add-service=mysql
firewall-cmd --reload
mysql_secure_installation
vim /etc/my.cnf
[mysqld]
server-id=2
replicate-wild-do-table=music.%
systemctl restart mariadb
mysql -e 'create database music'
mysql -u root -p music < music.sql
mysql -u root -p
# change master to
master_host='dbmaster',
master_user='repl',                 #configured on dbmaster side
master_password='secret',           #configured on dbmaster side
master_log_file='mysql-bin.000001', #show master status; on dbmaster
master_log_pos=245;                 #show master status; on dbmaster
$ start slave;
$ show processlist;
$ show master status\G;
```

dbmaster

```shell
mysql -u root -p
insert into music.hiphop(registration,singer,title,yt)values(4,'The Fugees','Ready Or Not','https://youtu.be/aIXyKmElvv8');
select * from music.hiphop;
# +-----------------+--------------------+------------------------------+--------------+
# | singer          | title              | yt                           | registration |
# +-----------------+--------------------+------------------------------+--------------+
# | DMX             | Ruff Ryders Anthem | https://youtu.be/ThlhSnRk21E |            1 |
# | Swollen Members | Full Contact       | https://youtu.be/iVZ1pvJnmsM |            2 |
# | Dead Prez       | Hip Hop            | https://youtu.be/4jNyr6BJZuI |            3 |
# | The Fugees      | Ready Or Not       | https://youtu.be/aIXyKmElvv8 |            4 |
# +-----------------+--------------------+------------------------------+--------------+
```

dbslave

```shell
select * from music.hiphop;
# +-----------------+--------------------+------------------------------+--------------+
# | singer          | title              | yt                           | registration |
# +-----------------+--------------------+------------------------------+--------------+
# | DMX             | Ruff Ryders Anthem | https://youtu.be/ThlhSnRk21E |            1 |
# | Swollen Members | Full Contact       | https://youtu.be/iVZ1pvJnmsM |            2 |
# | Dead Prez       | Hip Hop            | https://youtu.be/4jNyr6BJZuI |            3 |
# | The Fugees      | Ready Or Not       | https://youtu.be/aIXyKmElvv8 |            4 |
# +-----------------+--------------------+------------------------------+--------------+
```
