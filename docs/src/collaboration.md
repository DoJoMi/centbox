# Collaboration/Communication

------------------------------------------------------------------------

## Harbor

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2 vim wget
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce -y
systemctl enable docker && systemctl start docker
curl -L https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
wget https://storage.googleapis.com/harbor-releases/harbor-offline-installer-v1.5.2.tgz -O harbor.tgz
tar -xf habor.tar.gz
mkdir cert && cd cert
openssl req -sha256 -x509 -days 365 -nodes -newkey rsa:4096 -keyout  harbor.k8s4.fun.key -out harbor.k8s4.fun.crt
cd ~/
sed -i "s%hostname = reg.mydomain.com%hostname = harbor.k8s4.fun%" harbor/harbor.cfg
sed -i "s%ui_url_protocol = http%ui_url_protocol = https%" harbor/harbor.cfg
sed -i "s%ssl_cert = /data/cert/server.crt%ssl_cert = /root/cert/harbor.k8s4.fun.crt%" harbor/harbor.cfg
sed -i "s%ssl_cert_key = /data/cert/server.key%ssl_cert_key = /root/cert/harbor.k8s4.fun.key%" harbor/harbor.cfg
./harbor/install.sh --with-notary --with-clair --with-chartmuseum
docker-compose up -d
docker-compose down
docker ps -a
# CONTAINER ID        IMAGE                                       COMMAND                  CREATED             STATUS                             PORTS                                                              NAMES
# ab5d2159b558        vmware/nginx-photon:v1.5.2                  "nginx -g 'daemon of…"   21 seconds ago      Up 16 seconds (health: starting)   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp   nginx
# 3c0c594c73a5        vmware/harbor-jobservice:v1.5.2             "/harbor/start.sh"       21 seconds ago      Up 16 seconds                                                                                         harbor-jobservice
# fb206fc2bc2b        vmware/notary-server-photon:v0.5.1-v1.5.2   "/bin/server-start.sh"   23 seconds ago      Up 18 seconds                                                                                         notary-server
# fc917df5a4a7        vmware/harbor-ui:v1.5.2                     "/harbor/start.sh"       24 seconds ago      Up 20 seconds (health: starting)                                                                      harbor-ui
# afd08d4f47ee        vmware/clair-photon:v2.0.4-v1.5.2           "/docker-entrypoint.…"   25 seconds ago      Up 7 seconds (health: starting)    6060-6061/tcp                                                      clair
# 98a3b2fb4d1e        vmware/notary-signer-photon:v0.5.1-v1.5.2   "/bin/signer-start.sh"   26 seconds ago      Up 22 seconds                                                                                         notary-signer
# 5af15a99f265        vmware/harbor-adminserver:v1.5.2            "/harbor/start.sh"       28 seconds ago      Up 23 seconds (health: starting)                                                                      harbor-adminserver
# 793eac028d74        vmware/redis-photon:v1.5.2                  "docker-entrypoint.s…"   28 seconds ago      Up 24 seconds                      6379/tcp                                                           redis
# 11af6ff9c7fe        vmware/registry-photon:v2.6.2-v1.5.2        "/entrypoint.sh serv…"   28 seconds ago      Up 23 seconds (health: starting)   5000/tcp                                                           registry
# 38b78af74d10        vmware/harbor-db:v1.5.2                     "/usr/local/bin/dock…"   29 seconds ago      Up 24 seconds (health: starting)   3306/tcp                                                           harbor-db
# c41c2bc24487        vmware/postgresql-photon:v1.5.2             "/entrypoint.sh post…"   29 seconds ago      Up 24 seconds (health: starting)   5432/tcp                                                           clair-db
# 083ec0c64815        vmware/mariadb-photon:v1.5.2                "/usr/local/bin/dock…"   29 seconds ago      Up 25 seconds                      3306/tcp                                                           notary-db
# 67caaa14b127        vmware/harbor-log:v1.5.2                    "/bin/sh -c /usr/loc…"   29 seconds ago      Up 28 seconds (health: starting)   127.0.0.1:1514->10514/tcp                                          harbor-log

firefox https://harbor.k8s4.fun
# admin 
# Harbor12345

# adjust all relevant information inside harbor/harbor.cfg
```

## Gitlab

```shell
yum install epel-release certbot firewalld vim -y
systemctl enable firewalld && systemctl start firewalld
firewall-cmd --permanent --add-service={https,http}
#firewall-cmd --zone=public --remove-service=ssh --permanent
firewall-cmd --reload
curl -O curl -O https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh
chmod +x script.rpm.sh && ./script.rpm.sh
yum install gitlab-ce -y
gitlab-ctl reconfigure
mkdir -p /var/www/public/letsencrypt
cp /etc/gitlab/gitlab.rb /etc/gitlab/gitlab.rb.bak
cat >> /etc/gitlab/gitlab.rb <<eof
web_server['home'] = '/var/opt/gitlab/nginx'
nginx['custom_gitlab_server_config'] = "location ^~ /.well-known {
root /var/www/public/letsencrypt;
}"
eof
gitlab-ctl reconfigure
certbot certonly --webroot --webroot-path=/var/www/public/letsencrypt -d gitlab.k8s4.fun
sed -i "s%external_url 'http://gitlab.example.com'%external_url 'https://gitlab.k8s4.fun'%" /etc/gitlab/gitlab.rb
cat >> /etc/gitlab/gitlab.rb<<eof
nginx['redirect_http_to_https'] = true
nginx['ssl_certificate'] = "/etc/letsencrypt/live/gitlab.k8s4.fun/fullchain.pem"
nginx['ssl_certificate_key'] = "/etc/letsencrypt/live/gitlab.k8s4.fun/privkey.pem"
eof
gitlab-ctl reconfigure
# renew the certificate every month
crontab -e
0 2 1 * * /usr/bin/certbot renew --quiet --renew-hook "/usr/bin/gitlab-ctl restart nginx"
# backup to move to another server
tar -cvcf gitlab.letsencrypt.tar.gz /etc/letsencrypt/*

# gitlab-ci-runners - https://botleg.com/stories/setup-gitlab-for-docker-based-development/
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum makecache fast
yum install docker-ce -y
systemctl enable docker && systemctl start docker
usermod -aG docker dojomi
relogin
# as default user
docker run -d --name runner1 --restart always -v /var/run/docker.sock:/var/run/docker.sock gitlab/gitlab-runner:latest
docker run -d --name runner2 --restart always -v /var/run/docker.sock:/var/run/docker.sock gitlab/gitlab-runner:latest
docker exec -it runner1 gitlab-runner register
# executor = docker
# defaultDockerImage = docker:git
docker exec -it runner1 nano /etc/gitlab-runner/config.toml
# volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]

# container-registry
# as an alternative harbor could be used as well
certbot certonly --webroot --webroot-path=/var/www/public/letsencrypt -d registry.k8s4.fun
cat >> /etc/gitlab/gitlab.rb <<eof
registry_external_url 'https://registry.k8s4.fun'
registry_nginx['ssl_certificate'] = "/etc/letsencrypt/live/registry.k8s4.fun/fullchain.pem"
registry_nginx['ssl_certificate_key'] = "/etc/letsencrypt/live/registry.k8s4.fun/privkey.pem"
eof
cat >> /etc/hosts <<eof <IP> registry.k8s4.fun eof
gitlab-ctl reconfigure

# change root-user-pwd
gitlab-rails console production
user = User.where(id: 1).first
user.password = 'secret_pass'
user.password_confirmation = 'secret_pass'
user.save!
```

## Openfire XMPP/Jabber

```shell
echo "192.168.1.15 xmpp.labs.local xmpp" >> /etc/hosts
yum -y install mariadb-server mariadb
systemctl start mariadb; systemctl enable mariadb
mysql_secure_installation
mysql -u root -p
create database openfire;
grant usage on *.* to 'openfire'@'localhost' identified by 'openfire' with max_queries_per_hour 0 max_connections_per_hour 0 max_updates_per_hour 0 max_user_connections 0;
grant all on openfire.* to 'openfire'@'localhost' identified by 'openfire';
flush privileges;
vim /etc/my.cnf
[mysqld]
bind-address = 127.0.0.1
systemctl restart mysqld
yum install -y glibc.i686 wget
wget http://www.igniterealtime.org/downloadServlet?filename=openfire/openfire-4.0.2-1.i386.rpm
rpm -ivh downloadServlet?filename=openfire/openfire-4.0.2-1.i386.rpm
systemctl enable openfire; sytecmtl restart openfire; systemctl status openfire -l
# /usr/lib64/security/pam_fprintd.so: cannot open shared object file: No such file or directory
yum install -y pam_fprint
sytemctl restart openfire
firefox http://192.168.1.15:9090/setup/index.jsp
```

![](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/collaboration/1.png)

![](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/collaboration/2.png)

![](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/collaboration/3.png)

![](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/collaboration/4.png)

![](https://raw.githubusercontent.com/DoJoMi/centbox/master/docs/img/collaboration/5.png)
