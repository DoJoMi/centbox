## Authentication

------------------------------------------------------------------------

# OpenLDAP

```shell
yum install -y openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel firewalld
systemctl start firewalld && systemctl enable firewalld
mkdir /etc/openldap/ldif
chown ldap. /etc/openldap/ldif
cp /etc/openldap/ldap.conf /etc/openldap/ldap.conf.bak
cat > /etc/openldap/ldap.conf <<EOF
base    dc=k8s4,dc=fun
uri     ldap:///ldap.k8s4.fun
EOF
firewall-cmd --add-service=ldap --permanent
firewall-cmd --reload
systemctl enable slapd && systemctl start slapd

/usr/sbin/slappasswd -s admin -n > /etc/openldap/passwd
# pw: admin
# {SSHA}lufHuXC+s6i3hKzkwUgTFEcFfRTDqa58
cat > /etc/openldap/ldif/cn\=config_admin_olcRootPW.ldif <<EOF
dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}lufHuXC+s6i3hKzkwUgTFEcFfRTDqa58
EOF
ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/openldap/ldif/cn\=config_admin_olcRootPW.ldif

# basic schemas
cp -a /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown ldap. /var/lib/ldap/DB_CONFIG
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif

cat > /etc/openldap/ldif/cn\=config_baseDN.ldif <<EOF
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=k8s4,dc=fun
-
replace: olcRootDN
olcRootDN: cn=ldapadm,dc=k8s4,dc=fun

dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read by dn.base="cn=ldapadm,dc=k8s4,dc=fun" read by * none
EOF
ldapmodify -Y EXTERNAL -H ldapi:/// -f/etc/openldap/ldif/cn\=config_baseDN.ldif

cat > /etc/openldap/ldif/cn\=config_tree.ldif <<EOF
dn: dc=k8s4,dc=fun
dc: k8s4
o: k8s4.fun
objectClass: top
objectClass: domain

dn: cn=ldapadm,dc=k8s4,dc=fun
objectClass: organizationalRole
cn: ldapadm
description: ldapadm

dn: ou=People,dc=k8s4,dc=fun
objectClass: organizationalUnit
ou: People

dn: ou=Group,dc=k8s4,dc=fun
objectClass: organizationalUnit
ou: Group
EOF
ldapadd -x -D cn=ldapadm,dc=k8s4,dc=fun -W -f /etc/openldap/ldif/cn\=config_tree.ldif

# create new user
cat > /etc/openldap/ldif/cn\=config_tree.ldif <<EOF
dn: uid=dojomi,ou=People,dc=k8s4,dc=fun
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: dojomi
uid: dojomi
uidNumber: 9999
gidNumber: 100
homeDirectory: /home/dojomi
loginShell: /bin/bash
gecos: dojomi [Infrastructure]
userPassword: {crypt}x
shadowLastChange: 17058
shadowMin: 0
shadowMax: 99999
shadowWarning: 7
EOF
ldapadd -x -D cn=ldapadm,dc=k8s4,dc=fun -W -f /etc/openldap/ldif/cn\=config_tree.ldif

# assign pwd to the user
# -s specify the password for the username
# -x username for which the password is changed
# -D Distinguished name to authenticate to the LDAP server.
ldappasswd -s imojod -W -D "cn=ldapadm,dc=k8s4,dc=fun" -x "uid=dojomi,ou=People,dc=k8s4,dc=fun"

# verify
ldapsearch -H ldap://ldap.k8s4.fun:389 -x cn=dojomi -b dc=k8s4,dc=fun
# delete entry
ldapdelete -W -D "cn=ldapadm,dc=k8s4,dc=fun" "uid=dojomi,ou=People,dc=k8s4,dc=fun"

cat > /etc/openldap/ldap.conf <<EOF
base    dc=k8s4,dc=fun
uri     ldaps:///ldap.k8s4.fun
uri     ldapi:///ldap.k8s4.fun
ldap_version 3
ssl start_tls
ssl on
tls_checkpeer no
TLS_CACERTDIR /etc/openldap/certs
TLS_REQCERT allow
EOF
firewall-cmd --add-service=ldaps --permanent
firewall-cmd --reload
setsebool -P httpd_can_connect_ldap on

# create ssl-certificate
# openssl req -new -x509 -nodes -out /etc/openldap/certs/cert.pem -keyout /etc/openldap/certs/priv.pem -days 365
# chown ldap. /etc/openldap/certs/*
# chmod 600 /etc/openldap/certs/priv.pem
# or could be handled as well with
cd /etc/pki/tls/certs 
make server.key 
openssl rsa -in server.key -out server.key 
make server.csr 
openssl x509 -in server.csr -out server.crt -req -signkey server.key -days 3650
cp /etc/pki/tls/certs/server.key /etc/pki/tls/certs/server.crt /etc/pki/tls/certs/ca-bundle.crt  /etc/openldap/certs/
chown ldap. /etc/openldap/certs/server.key /etc/openldap/certs/server.crt /etc/openldap/certs/ca-bundle.crt 

sed -i 's|SLAPD_URLS="ldapi:\/\/\/ ldap:\/\/\/"|SLAPD_URLS="ldapi:\/\/\/ ldap:\/\/\/ ldaps:\/\/\/"|' /etc/sysconfig/slapd

cat > /etc/openldap/ldif/cn\=config_baseDN.ldif <<EOF
dn: cn=config
changetype: modify
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/openldap/certs/ca-bundle.crt
-
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/server.crt
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/server.key
-
replace: olcIdleTimeout
olcIdleTimeout: 30
-
replace: olcTimeLimit
olcTimeLimit: 15
-
replace: olcReferral
olcReferral: ldaps://ldap.k8s4.fun
-
replace: olcLogLevel
olcLogLevel: stats

EOF
ldapmodify -Y EXTERNAL -H ldapi:/// -f/etc/openldap/ldif/cn\=config_baseDN.ldif
slaptest -u
systemctl restart slapd

# check from external-host
#-x = Use simple authentication instead of SASL
#-D = Use the Distinguished Name binddn to bind to the LDAP directory
#-w = Use passwd as the password for simple authentication
#-b = Use searchbase as the starting point for the search instead of the default
#-H = Specify URI(s) referring to the ldap server(s)

openssl s_client -connect ldap.k8s4.fun:636
ldapsearch -H ldaps://ldap.k8s4.fun:636 -x cn=ldapadm -b dc=k8s4,dc=fun
ldapsearch -D "cn=ldapadm" -W -p 389 -h ldap.k8s4.fun -b "dc=k8s4,dc=fun" -s sub -x -ZZ "(objectclass=*)"
ldapsearch -w admin -x -h ldap.k8s4.fun -D cn=config -b cn=config "(objectclass=olcGlobal)"
```
