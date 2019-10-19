# Configuration-Management

------------------------------------------------------------------------

## Ansible

```shell
# master
nmcli general hostname amaster.labs.local
systemctl restart systemd-hostnamed
yum install -y epel-release
sudo yum install -y ansible
echo "192.168.1.11 amaster.labs.local amaster" >> /etc/hosts
echo "192.168.1.12 aclient.labs.local aclient" >> /etc/hosts
```

```shell
# node
nmcli general hostname aclient.labs.local
systemctl restart systemd-hostnamed
echo "192.168.1.11 amaster.labs.local amaster" >> /etc/hosts
echo "192.168.1.12 aclient.labs.local aclient" >> /etc/hosts
```

```shell
# master
ssh-keygen -t rsa -b 4096 -C "aclient@labs.local" -f /root/.ssh/aclient_rsa
ssh-copy-id -i ~/.ssh/aclient_rsa.pub root@192.168.1.12
# test it out
ssh root@IP -i ~/.ssh/aclient_rsa
# add remote host to /etc/ansible/hosts
cp -rv /etc/ansible/hosts /etc/ansible/hosts.bak
cat /dev/null > /etc/ansible/hosts
cat >/etc/ansible/hosts<<eof
[aclient]
192.168.1.18 
eof
ansible -m ping aclient --private-key=~/.ssh/aclient
ansible -m command -a 'df -h' aclient
ansible -m command -a 'uptime' aclient
```

```shell
# Ansible Sample-Playbook
# master
mkdir ~/ansible_playbooks && touch ~/ansible_playbooks/httpd.yml 
cat > ~/ansible_playbooks/httpd.yml << eof
---
- hosts: aclient
vars:
    http_port: 80
  remote_user: root
  tasks:
  - name: ensure latest version is running
    yum: name=httpd state=latest
  - name: enable httpd
    service: name=httpd state=started enabled=yes
  - name: add port to firewalld
    firewalld: service=http permanent=true state=enabled
  - shell: firewall-cmd --reload
  handlers:
    - name: restart apache
      service: name=httpd state=restarted
eof
ansible-playbook ~/ansible_playbooks/httpd.yml --private-key=~/.ssh/aclient_rsa
```

```shell
# node
systemctl status httpd
firewall-cmd --list-all
firefox 192.168.1.12
```

## Puppet

```shell
# master
nmcli general hostname pmaster.labs.local
systemctl restart systemd-hostnamed
rpm -ivh https://yum.puppetlabs.com/el/7/products/x86_64/puppetlabs-release-7-11.noarch.rpm
yum install -y ntp
sed -i "s/^#restrict/restrict/" /etc/ntp.conf
cat >> /etc/ntp.conf <<eof
logfile /var/log/ntp.log
eof
firewall-cmd --add-service=ntp --permanent
firewall-cmd --reload
systemctl enable ntpd.service; systemctl start ntpd.service
ntpq -p
yum install -y puppet-server
echo "192.168.1.11 pmaster.labs.local pmaster" >> /etc/hosts
echo "192.168.1.12 pclient.labs.local pclient" >> /etc/hosts
systemctl enable puppetmaster.service; systemctl start puppetmaster.service
puppet resource service puppetmaster ensure=running enable=true
cat >> /etc/puppet/puppet.conf <<eof
[master]
    certname=pmaster.labs.local
eof
systemctl restart puppetmaster.service
netstat -ntlp
firewall-cmd --permanent --add-port=8140/tcp
firewall-cmd --reload
```

```shell
# node
nmcli general hostname pclient.labs.local
systemctl restart systemd-hostnamed
rpm -ivh https://yum.puppetlabs.com/el/7/products/x86_64/puppetlabs-release-7-11.noarch.rpm
yum install -y puppet
echo "192.168.1.11 pmaster.labs.local pmaster" >> /etc/hosts
echo "192.168.1.12 pclient.labs.local pclient" >> /etc/hosts
vim /etc/puppet/puppet.conf
[main]
    server=pmaster.labs.local
ZZ
systemctl enable puppet.service; systemctl start puppet.service
```

```shell
# master
puppet cert list -a
puppet cert sign pclient.labs.local

# Error: Could not find certificate request for pclient.labs.local
# node   --> puppet agent --master 192.168.1.11 --waitforcert 60 --test --verbose
# follow instructions to revoke the certificate and resign a new certificate
# master --> puppet cert clean pclient.labs.local
# node   --> find /var/lib/puppet/ssl -name pclient.labs.local.pem -delete
# node   --> systemctl restart puppet
# master --> puppet cert list --all
# master --> puppet cert sign pclient.labs.local
```

```shell
# Puppet Sample
# master
touch /etc/puppet/manifest/site.pp
cat > /etc/puppet/manifest/site.pp <<eof
node 'pclient.labs.local'{
    package{'httpd': ensure => installed}
    service{'httpd': ensure => running, enable => true}
}
eof
```

```shell
# node
puppet agent -tv
systemctl status httpd
```

## Saltstack

```shell
# master
nmcli general hostname salt-master.labs.local
systemctl restart systemd-hostnamed
install https://repo.saltstack.com/yum/redhat/salt-repo-latest-2.el7.noarch.rpm
yum clean expire-cache
yum install salt-master
echo "192.168.1.11 smaster.labs.local smaster" >> /etc/hosts
echo "192.168.1.12 sminion.labs.local sminion" >> /etc/hosts
cat > /etc/salt/master << eof
interface: 192.168.1.11
hash_type: sha256
eof
firewall-cmd --get-active-zones
firewall-cmd --permanent --zone=public --add-port=4505-4506/tcp
firewall-cmd --reload
firewall-cmd --zone=public --list-ports
systemctl start salt-master.service
systemctl enable salt-master.service
```

```shell
# minion
nmcli general hostname sminion.labs.local
systemctl restart systemd-hostnamed
yum install https://repo.saltstack.com/yum/redhat/salt-repo-latest-2.el7.noarch.rpm
yum clean expire-cache
yum install salt-minion
cat > /etc/salt/minion << eof
master: 192.168.1.11
hash_type: sha256
eof
systemctl start salt-minion.service
systemctl enable salt-minion.service
echo "192.168.1.11 smaster.labs.local smaster" >> /etc/hosts
echo "192.168.1.12 sminion.labs.local sminion" >> /etc/hosts
```

```shell
# master
salt-key -L
salt-key --accept=salt-minion
salt 'salt-minion' cmd.run pwd
```

```shell
# Saltstack Sample with commandline
# master
salt 'salt-minion' user.add dojomi
salt 'salt-minion' user.chgroups dojomi admin append=True
salt 'salt-minion' shadow.set_password dojomi '$6$3NSRBRZ.$nehEib9TuHvHmqoRdi3Ouve7mcG2sDVA1XXPGPh1PsMW6webiZGq7JOlnWYhmyg2mKUpufETbjr0KyMu6YFOz/'
salt 'salt-minion' user.chshell dojomi /bin/bash
```
