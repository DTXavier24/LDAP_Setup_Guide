<h1 align="center">OpenLDAP Configuration Guide 🌐</h1>

<p align="center">
  <img alt="Version" src="https://img.shields.io/badge/version-1.0.0-blue.svg?cacheSeconds=2592000" />
  <img alt="Distributions" src="https://img.shields.io/badge/Distributions-CentOS%20%7C%20Ubuntu-green.svg" />
  <img alt="License: MIT" src="https://img.shields.io/badge/License-MIT-yellow.svg" />
</p>

<p align="center">
  Comprehensive Guide for Configuring OpenLDAP on Linux Systems
</p>

> This repository provides a detailed walkthrough for configuring open-source LDAP (Lightweight Directory Access Protocol) on CentOS and Ubuntu Linux distributions. Learn how to integrate your systems with centralized authentication services efficiently and securely.

## ✨ Features
- Step-by-step OpenLDAP server and client configuration
- Support for CentOS and Ubuntu
- Secure authentication setup with SSL/TLS
- Troubleshooting and best practices

## 📘 References
- https://www.ibm.com/docs/en/rpa/23.0?topic=ldap-installing-configuring-openldap#installing-openldap-on-linux
- https://ubuntu.com/server/docs/how-to-set-up-sssd-with-ldap
- https://www.youtube.com/watch?v=Q1kwHHmlxnQ

## 🔧 Prerequisites
- Ensure both server and client can ping each other
- Ensure server machine has DNS configured and working properly.

## 🔨	 Default Installation

### On Centos 
1. Include EPEL repository
<br>Due to dnf repository doesn't include openldap packages, we'll need to get it from different reposiroty such as epel.
```console
sudo dnf install epel-release
```

2. Start install the packages.
```console
sudo dnf --enablerepo=epel -y install openldap-servers openldap-clients
```

3. Enable system services.
```console
sudo systemctl enable --now slapd
sudo systemctl start slapd
```

4. Make changes to the firewall for external connections.
```console
sudo firewall-cmd --permanent --add-port=389/tcp --add-port=389/udp
sudo firewall-cmd --reload
sudo setsebool -P allow_ypbind=1 authlogin_nsswitch_use_ldap=1
sudo setsebool -P httpd_can_connect_ldap on
```

5. Make changes to default configurations.
```console
sudo nano /etc/openldap/ldap.conf
```

Uncomment these lines and change the base names accoprding to the hostname.
```console
#BASE     dc=example,dc=com
#URI      ldap://ldap.example.com ldap://ldap-master.example.com:666
```
to 
```console
BASE     dc=snaserver,dc=sna,dc=org
URI      ldap://snaserver.sna.org
```

![image](https://github.com/user-attachments/assets/a3184f61-18ea-46ec-acab-00f0a0499cec)

![image](https://github.com/user-attachments/assets/21051ec3-2281-4afb-b7cb-0b9effd166f2)

6. Create root user
Create a directory to store LDAP configurations files.
```console
mkdir ldap_conf
cd ldap_conf
```

Create a password hash.
```console
slappasswd > password.txt
```

Copy the password.
```console
cat password.txt
```

Create a new file for root user configuration.
```console
nano rootpw.ldif
```

Paste the following below inside the file. Change olcRootPW according to the password gotten above.
```console
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}yh/GrT7AsObYUoHu89ynjzOljpBP10sp
```

Enter the file as configurations into LDAP
```console
ldapadd -Y EXTERNAL -H ldapi:/// -f rootpw.ldif
```

![image](https://github.com/user-attachments/assets/2dfd7a34-b4b9-4a2d-ba4a-785f9e2c1286)

Import the LDAP schema.
```console
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/openldap.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/dyngroup.ldif
```

Now, we'll be creating a user who is the admin for the server. This admin will have all rights to the server such as modifications.

Create a new file for manager configuration.
```console
nano manager.ldif
```

Paste the following below into the file.
- modify olcSuffix and olcRootDN according to the base name given above.
- modify olcRootPW according to get password hash gotten above.
```console
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=snaserver,dc=sna,dc=org

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=Manager,dc=snaserver,dc=sna,dc=org

dn: olcDatabase={2}mdb,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}yh/GrT7AsObYUoHu89ynjzOljpBP10sp
```

![image](https://github.com/user-attachments/assets/3685b3d4-bb97-46b1-acdb-9e84be6d6993)

Enter the file as configurations into LDAP
```console
ldapadd -Y EXTERNAL -H ldapi:/// -f manager.ldif
```

Create the structure for LDAP organization.
```console
nano org.ldif
```

Paste the following below inside the file.
modify each 'dn' according to the base name given above
```console
dn: dc=snaserver,dc=sna,dc=org
objectClass: top
objectClass: dcObject
objectclass: organization
o: LDAP Server
dc: snaserver

dn: cn=Manager,dc=snaserver,dc=sna,dc=org
objectClass: organizationalRole
cn: Manager
description: LDAP Manager

dn: ou=People,dc=snaserver,dc=sna,dc=org
objectClass: organizationalUnit
ou: People

dn: ou=Group,dc=snaserver,dc=sna,dc=org
objectClass: organizationalUnit
ou: Group
```

![image](https://github.com/user-attachments/assets/4e250fef-4fcb-4270-af43-9d0037f89f6a)

Enter the file as configurations into LDAP using 'Manager' credentials.
```console
ldapadd -x -D cn=Manager,dc=snaserver,dc=sna,dc=org -W -f org.ldif
```

7. Register New User
Create a file for user's configuration
```console
nano addUser.ldif
```

Paste the following below inside the file.
- modify each 'dn' according to the base name given above.
- modify userPassword according to the password hash above.
- always change uidNumber and gidNumber when creating new users.
```Console
dn: uid=Jack,ou=People,dc=snaserver,dc=sna,dc=org
changetype: add
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: Jack
cn: Jack
sn: Jack
displayName: Jack
mail: jack@snaserver.sna.org
userPassword: {SSHA}xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
loginShell: /bin/bash
uidNumber: 20001
gidNumber: 20001
homeDirectory: /home/jack

dn: cn=Jack,ou=Group,dc=snaserver,dc=sna,dc=org
objectClass: posixGroup
cn: Jack
gidNumber: 20001
memberUid: Jack
```

![image](https://github.com/user-attachments/assets/b8c72cb3-14c4-4d89-baf7-afa3a127ac5d)

Enter the file as configurations into LDAP using "Manager" credentials
```console
ldapadd -D "cn=Manager,dc=snaserver,dc=sna,dc=org" -W -f addUser.ldif
```

Confirm that the server is working properly with the 2 commands below.
```console
ldapsearch -x -LLL uid=Jack
or
ldapsearch -x -LLL uid=* uid

ldapsearch -x -H ldap://192.168.100.4 -b "dc=snaserver,dc=sna,dc=org"
```
![image](https://github.com/user-attachments/assets/1476192a-5905-41ba-8445-0d5e289ee135)

![image](https://github.com/user-attachments/assets/cd9ea616-aa61-4d22-906c-53837c110c79)

This means LDAP is working as intended.

### On Ubuntu
1. Install sssd with some other packages.
>SSSD (System Security Services Daemon) is a powerful service that provides authentication, authorization, and account management for Linux system. SSSD acts as an intermediary between Linux systems and different identity and authentication providers.
```console
sudo apt install sssd-ldap ldap-utils
```

2. Modify default configurations
Create configuration file.
```console
sudo nano /etc/sssd/sssd.conf
```

Paste the following below inside the file.
```console
[sssd]
config_file_version = 2
domains = snaserver.sna.org

[domain/snaserver.sna.org]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://192.168.100.4
cache_credentials = True
ldap_search_base = dc=snaserver,dc=sna,dc=org
```

Change file permission
```console
sudo chmod 0600 /etc/sssd/sssd.conf
```

Change file ownership
```console
sudo chwon root:root /etc/sssd/sssd.conf
```

3. Start sssd services
```console
sudo systemctl start sssd.service
```

4. Enable PAM to create home directories
```console
sudo pam-auth-update --enable mkhomedir
```

5. Verify LDAP server
<br>Check if user exists in /etc/passwd
```console
getent passwd Jack
```

Check if user id exists
```console
id Jack
```

Try login as Jack
```console
sudo login Jack
```

![image](https://github.com/user-attachments/assets/3f62776f-0332-48d5-adcf-d91c88295b47)

This means LDAP is working as intended

## 🔏 Secure OpenLDAP with SSL/TLS

### On Centos
1. Setup hostname
```console
sudo nano /etc/hosts
```

Add respective IP address and hostname into the file
```console
192.168.100.4  snaserver.sna.org
```

2. Create an SSL certificate using OpenSSL
```console
sudo dnf -y install openssl
```

Create a directory to store the certificate
```console
sudo mkdir /etc/ssl/ldap
```

Create the .key and .crt file
```console
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/ldap/ldap-selfsigned.key -out /etc/ssl/ldap/ldap-selfsigned.crt
```

![image](https://github.com/user-attachments/assets/7da40590-e75f-4cc9-826d-f0be44b9ed1c)

Change both file owner to ldap
```console
sudo chown ldap:ldap /etc/ssl/ldap/ldap-selfsigned.key
sudo chown ldap:ldap /etc/ssl/ldap/ldap-selfsigned.crt
```

3. Make OpenLDAP configuration changes
<br>Create a file to make config changes.
```console
sudo nano ssl.ldif
```

Paste the following into the file.
```console
dn: cn=config
changetype: modify
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ssl/ldap/ldap-selfsigned.crt
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ssl/ldap/ldap-selfsigned.crt
```

Use ldapmodify to make configuration changes.
```console
ldapmodify -Y EXTERNAL -H ldapi:/// -f ssl.ldif
```

Change URI reference in OpenLDAP configuration file.
```console
sudo nano /etc/openldap/ldap.conf
```

On the 'URI' line, make sure it's using 'ldaps' instead of 'ldap'
```console
URI  ldaps://192.168.100.4
```

On the 'TLS_CACERT' line, uncomment it and mention the SSL certificate path
```console
TLS_CACERT  /etc/ssl/ldap/ldap-selfsigned.crt
```

Restart the service.
```console
sudo systemctl restart slapd
```

4. Make firewall changes
```console
sudo firewall-cmd --permanent --add-port=636/tcp
sudo firewall-cmd --reload
```

5. Test the service
<br>Run the command below and make sure no errors occur.
```console
ldapsearch -x -H ldaps://192.168.100.4 -b "dc=snaserver,dc=sna,dc=org"
```
```console
ldapsearch -x -H ldaps://snaserver.sna.org -b "dc=snaserver,dc=sna,dc=org"
```

![image](https://github.com/user-attachments/assets/007bcc6c-f853-4fe0-9c7e-92c5490190f0)

This means ldaps is working as intended.

### On Ubuntu
1. Allow ldap to request SSL certificate from server
```console
sudo nano /etc/ldap/ldap.conf
```

Add the following below
```console
TLS_REQCERT  allow
```

2. Change sssd configurations
```console
sudo nano /etc/sssd/sssd.conf
```

Add the following below.
```console
ldap_id_use_start_tls = true
```

On the 'ldap_uri', make sure it's using 'ldaps'
```console
ldap_uri = ldaps://snaserver.sna.org
```

Restart SSSD service
```console
sudo systemctl restart sssd
```

3. Test the service
<br>Run the command below and make sure no errors occur.
```console
ldapsearch -x -H ldaps://snaserver.sna.org -b "dc=snaserver,dc=sna,dc=org"
```

![image](https://github.com/user-attachments/assets/9eeb4dd2-6847-4a0a-ab09-4c8f28d4a429)

This means ldaps is working as intended.

login as any ldap user
```console
sudo login Jack
```

Run this command
```console
ldapwhoami -x -ZZ -H ldap://snaserver.sna.org
```

On CentOS, run this command
Run this command
```console
sudo systemctl status slapd
```

![image](https://github.com/user-attachments/assets/6fa464c1-3ba9-4770-87be-b18ad3559bcc)

Make sure able to see log info such as 'STARTTLS', 'TLS established' and 'err=0'.

This means ldaps is working as intended.

Back to ubuntu, run this command.
```console
ldapwhoami -x -H ldaps://snaserver.sna.org
```

On CentOS, run this command
Run this command
```console
sudo systemctl status slapd
```

![image](https://github.com/user-attachments/assets/ddc69c3b-dc4e-4aa2-95d2-5593d3614468)

Make sure able to see log info such 'TLS established' and 'err=0'.

This means ldaps is working as intended.
