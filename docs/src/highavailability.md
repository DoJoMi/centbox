# High-Availability

------------------------------------------------------------------------

## Pacemaker/Corosync

basic configuration node1

```shell
nmcli general hostname centvm1
systemctl restart systemd-hostnamed
echo "192.168.1.11 node1.linuxhausen.net centvm1" >> /etc/hosts
echo "192.168.1.12 node2.linuxhausen.net centvm2" >> /etc/hosts
```

basic configuration node2

```shell
nmcli general hostname centvm2
systemctl restart systemd-hostnamed
echo "192.168.1.11 node1.linuxhausen.net centvm1" >> /etc/hosts
echo "192.168.1.12 node2.linuxhausen.net centvm2" >> /etc/hosts
```

node1

```shell
yum install -y pacemaker pcs
systemctl enable pcsd.service
systemctl start pcsd.service
systemctl enable pacemaker.service
systemctl start pacemaker.service
passwd hacluster
firewall-cmd --permanent --add-service=high-availability
firewall-cmd --reload
```

node2

```shell
yum install -y pacemaker pcs
systemctl enable pcsd.service
systemctl start pcsd.service
systemctl enable pacemaker.service
systemctl start pacemaker.service
passwd hacluster
firewall-cmd --permanent --add-service=high-availability
firewall-cmd --reload
```

node1

```shell
pcs cluster auth node1.linuxhausen.net node2.linuxhausen.net -u hacluster
pcs cluster setup --name HACLUSTER node1.linuxhausen.net node2.linuxhausen.net --force
pcs cluster enable --all
pcs cluster start --all
pcs status
pcs status corosync
yum install -y httpd
systemctl enable httpd
systemctl start httpd
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
cat > /etc/httpd/conf.d/server-status.conf <<eof
<Location /server-status>
    SetHandler server-status
    Order Deny,Allow
    Deny from all
    Allow from 127.0.0.1
</Location>
eof
$ pcs property set stonith-enabled=false
$ pcs property set no-quorum-policy=ignore
$ pcs property set default-resource-stickiness="INFINITY"
$ pcs property list
$ pcs resource create VirtIP IPaddr2 ip=192.168.1.100 cidr_netmask=24 op monitor interval=30s
$ pcs resource
$ cat > /var/www/html/index.html <<eof
<html>
 <head>     
    <title>HaCluster</title>
 <head>
 <body>
    <h1>node1.linuxhausen.net<h1>
 <body>
<html>                    
eof
```

node2

```shell
$ yum install -y httpd
$ systemctl enable httpd
$ systemctl start httpd
$ firewall-cmd --permanent --add-service=http
$ firewall-cmd --reload

$ cat > /var/www/html/index.html <<eof
<html>
 <head>     
    <title>HaCluster</title>
 <head>
 <body>
    <h1>node2.linuxhausen.net<h1>
 <body>
<html>                    
eof
```

node1

```shell
$ pcs resource create HTTPD apache configfile="/etc/httpd/conf/httpd.conf" statusurl="http://127.0.0.1/server-status" op monitor interval=30s
$ pcs constraint colocation add Httpd with VirtIP INFINITY
$ pcs constraint order VirtIP then Httpd
$ pcs status
$ firefox http://192.168.1.100/
$ pcs cluster stop node1.linuxhausen.net
$ pcs status
$ pcs cluster start node1.linuxhausen.net
$ pcs cluster stop --all
$ pcs cluster destroy
```

## Nginx-Loadbalancer

node1

```shell
nmcli general hostname centvm1.kubeops.rocks
systemctl restart systemd-hostnamed
echo "192.168.1.11 node1.kubeops.rocks centvm1" >> /etc/hosts
echo "192.168.1.12 node2.kubeops.rocks centvm2" >> /etc/hosts
echo "192.168.1.13 node3.kubeops.rocks centvm3" >> /etc/hosts
dnf install -y httpd
systemctl enable httpd; systemctl start httpd
firewall-cmd --permanent --add-service http
firewall-cmd --reload
cat > /var/www/html/index.html << eof
<html>
<head>
    <title>Loadbalancing with NGINX</title>
</head>
<body>
    <h1>centvm1</h1>
</body>
</html>
eof

curl http://localhost
```

node2

```shell
nmcli general hostname centvm2.kubeops.rocks
systemctl restart systemd-hostnamed
echo "192.168.1.11 node1.kubeops.rocks centvm1" >> /etc/hosts
echo "192.168.1.12 node2.kubeops.rocks centvm2" >> /etc/hosts
echo "192.168.1.13 node3.kubeops.rocks centvm3" >> /etc/hosts
dnf install -y httpd
systemctl enable httpd; systemctl start httpd
firewall-cmd --permanent --add-service http
firewall-cmd --reload
cat > /var/www/html/index.html << eof
<html>
<head>
    <title>Loadbalancing with NGINX</title>
</head>
 <body>
<h1>centvm2</h1>
</body>
</html>
eof

systemctl reload nginx
curl http://localhost
```

node3

```shell
dnf install -y nginx
systemctl enable nginx; systemctl start nginx
firewall-cmd --permanent --add-service http
firewall-cmd --reload
cat > /etc/nginx/nginx.conf<<eof
http{
    upstream lbsite{
        server node1.kubeops.rocks;
        server node2.kubeops.rocks;
    }
server{
    location / {
        prox_pass http://lbmysite;
    }
}
}
eof

systemctl reload nginx
curl http://localhost
```
