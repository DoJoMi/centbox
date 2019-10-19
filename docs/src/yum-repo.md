# Package/Repository-Management

------------------------------------------------------------------------

## Package Manager

```shell
|:-------------------------------------------------------------------:|
|                           rpm                                       |
|:-------------------------------------------------------------------:|
| rpm -qi <pkg>.rpm         | -query information                      |
| rpm -ivh <pkg>.rpm        | -install -verbose -headway              |
| rpm -Uvh <pkg>.rpm        | -update -verbose -headway               |
| rpm -Fvh <pkg>.rpm        | -upgrade -verbose -headway              |
|:-------------------------------------------------------------------:|
|                        yum/yum                                      |
|:-------------------------------------------------------------------:|
| yum install -y <pkg>      | -install yes                            |
| yum remove <pkg>          | -remove                                 |
| yum update <pkg>          | -update                                 |
| yum search <pkg>          | -search                                 |
| yum info <pkg>            | -info                                   |
| yum list installed|less   | -list installed packages                |
|:-------------------------------------------------------------------:|
```

## Network Repository

```shell
# web availability
yum install httpd
firewall-cmd --permanent --add-service http
firewall-cmd --reload
systemctl enable httpd; systemctl start httpd
mkdir /var/www/html/repo/
rsync -avz rsync://centos.mirroraustria.at/CentOS/7.2.1511/os/x86_64/ /var/www/html/repo/
cp -rv /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf.bak
cat >> /etc/httpd/conf/httpd.conf <<eof
Alias /repo /var/www/html/repo/
<Directory "/var/www/html/repo/">
 Options Indexes
 AllowOverride None
</Directory>
eof
systemctl restart httpd
firefox 192.168.1.11/repo
```

```shell
# client availability
cat > /etc/yum.repos.d/labs.local.repo <<eof
[Labs.LocalRepo]
name=labs.local CentOS Repo
baseurl= 'http://192.168.1.11/repo'
enabled=1
gpgcheck=1
gpgkey= 'http://192.168.1.11/repo/RPM-GPG-KEY-CentOS-7'
eof
yum clean all
yum install -y links
```

```shell
# keeping up2date
crontab -e
30 2 * * * rsync -avz rsync://centos.mirroraustria.at/CentOS/7.2.1511/os/x86_64/ /var/www/html/repo/
```
