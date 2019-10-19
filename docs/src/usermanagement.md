# User-Management

------------------------------------------------------------------------

## ACL

```shell
groupadd developer
groupadd admin
useradd -s /bin/bash -m -d /home/dojomi -c "dojomi" -g developer dojomi
passwd dojomi
useradd -s /bin/bash -m -d /home/tom -c "tom" -g admin tom
passwd tom
# permissions automatically created by useradd
chown dojomi:developer /home/dojomi
chown tom:admin /home/tom
chmod 1700 /home/dojomi
chmod 1700 /home/tom
# drwx------.
# drwx-----T.
vim /etc/fstab
# add acl after defaults 
# UUID=...    /home    ext4   defaults,acl   0   2
mount -o remount /
# srv --> generic share directory
mkdir -p srv/foo/bar
touch /foo/public.txt
touch /foo/bar/private.txt
# /srv
# └── foo
#     ├── bar
#     │   └── private.txt
#     └── public.txt

ls -lR foo/
# foo/:
# total 4
# drwxr-xr-x. 2 root root 4096 Feb 15 10:26 bar
# -rw-r--r--. 1 root root    0 Feb 15 10:26 public.txt
# foo/bar:
# total 0
# -rw-r--r--. 1 root root 0 Feb 15 10:26 private.txt

chmod -R 700 foo
# foo/:
# total 4
# drwx------. 2 root root 4096 Feb 15 10:26 bar
# -rwx------. 1 root root    0 Feb 15 10:26 public.txt
# foo/bar:
# total 0
# -rwx------. 1 root root 0 Feb 15 10:26 private.txt 

# /srv
# └── foo
#     ├── bar
#     │   └── private.txt --> admin
#     └── public.txt      --> developer&admin

# admin
# Rdm = --recursive --default -modify
setfacl -Rdm g:admin:rwx foo/
# for the existing files
setfacl -Rm g:admin:rwx foo/
# developer
setfacl -dm g:developer:rwx foo
setfacl -m  g:developer:rwx foo
setfacl -m  g:developer:rwx foo/public.txt

su - dojomi
# can't switch into bar because of permissions denied   
# /srv/
# └── foo
#     ├── bar [error opening dir]
#     └── public.txt

getfacl /srv/foo/bar
# file: srv/foo/bar/
# owner: root
# group: root
# user::rwx
# group::---
# group:admin:rwx
# mask::rwx
# other::---
# default:user::rwx
# default:group::---
# default:group:admin:rwx
# default:mask::rwx
# default:other::---

su - tom
# can switch in all folders and open files
# /srv
# └── foo
#     ├── bar
#     │   └── private.txt
#     └── public.txt

# remove the acl 
setfacl -b /srv/
```

## Shell limitation

```shell
ln -s /bin/bash /opt/rbash
useradd -s /opt/rbash -m -d /home/dojomi -c "dojomi" -g developer dojomi
passwd dojomi
mkdir /home/dojomi/bin
ln -s /bin/ping /home/dojomi/bin/ping
ln -s /bin/hostname /home/dojomi/bin/hostname
chown root. /home/dojomi/.bash_profile
chmod 755 /home/dojomi/.bash_profile
echo "export PATH=$HOME/bin" >> /home/dojomi/.bash_profile

su - dojomi
hostname
mkdir
# -rbash: mkdir: restricted
ping localhost
```

## Password Expiration

```shell
# PASS_MAX_DAYS 99999 --> 60 for all new users
sed -i "s/99999/60/" /etc/login.defs

# or use chage [-m mindays] [-M maxdays] [-d lastday] [-I inactive] [-E expiredate] [-W warndays] username
chage -M 60 dojomi

chage --list dojomi
# Last password change                                    : Feb 18, 2016
# Password expires                                        : Apr 18, 2016
# Password inactive                                       : never
# Account expires                                         : never
# Minimum number of days between password change          : 0
# Maximum number of days between password change          : 60
# Number of days of warning before password expires       : 7
```

Skel
----

```shell
# All new users get the same initial settings.
# /etc/skel is used by useradd to create the default settings

# edit /etc/default/useradd to change default settings
# default config
# GROUP=100
# HOME=/home/users
# INACTIVE=-1
# EXPIRE=
# SHELL=/bin/bash
# SKEL=/etc/skel
# CREATE_MAIL_SPOOL=yes

ls -A /etc/skel 
# .bash_profile  .bashrc  .maildir  .screenrc  .tcsh.config
```

**Make \~/.bashrc available for all new created users**
<https://github.com/DoJoMi/dotbash>

## Privileges

```shell
# create a new root named admin
useradd admin
passwd admin
usermod -G wheel admin
# switch from admin2root
vi /etc/pam.d/su
# mail forwarding
vi /etc/aliases
root:admin
newaliases
```

```shell
# allow only members of wheel group to use su command
useradd dojomi
passwd dojomi
sudo -l dojomi
su -
# not allowed
usermod -G wheel dojomi
cat >> /etc/pam.d/su <<eof
authrequired pam_wheel.so use_uid
eof
sudo -l dojomi
su -
# mail forwarding
sed -i "s/#root: marc/root: root dojomi"
newaliases
```

```shell
# reduce user some priviliges
visudo
# Cmnd_Alias SHUTDOWN = /sbin/halt, /sbin/shutdown, /sbin/poweroff, /sbin/reboot, /sbin/init
dojomi ALL=(ALL) ALL,!SHUTDOWN
```

```shell
# log sudo commands into a file
visudo
Defaults syslog=local1
```

```shell
vi /etc/rsyslog.conf
# line 42 or 46: add like follows
*.info;mail.none;authpriv.none;cron.none;local1.none /var/log/messages
local1.*                                             /var/log/sudo.log
systemctl restart rsyslog
```