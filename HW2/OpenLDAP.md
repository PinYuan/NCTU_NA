# OpenLDAP

### Install OpenLDAP

- Install packages

  `yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel`

- Start the LDAP service and enable it for the auto start of service on system boot.

  ```
  systemctl start slapd
  systemctl enable slapd
  ```

- Verify the LDAP.

  ```
  ss -antup | grep -i 389
  ```



## Firewall

Both master and slave

```
firewall-cmd --permanent --add-service=ldap
firewall-cmd --reload
```



### Setup LDAP admin password

- create an LDAP root password (replace ldppassword with your password).

  ```
  slappasswd -h {SSHA} -s hahaYouCatchMe
  ```

  The above command will generate an encrypted hash of entered password which you need to use in LDAP configuration file. So make a note of this and **keep it aside**.

- Output

  - Master {SSHA}SwXqgAoIBR38TN7FBNVJh26Pf1zB3Y0M
  - Slave {SSHA}W9ECJV5kTvYsIsCmOvls3mkKU1PcBtoD



### Configure OpenLDAP server

- configuration files are found in `/etc/openldap/slapd.d/`

- **All ldif file you created not put in `/etc/openldap/slapd.d/cn=config/`**

- We need to update these variables: olcSuffix, olcRootDN, olcRootPW

  - **olcSuffix** – Database Suffix, it is the **domain name** for which the LDAP server provides the information. In simple words, it should be changed to your domain name.

  - **olcRootDN** – **Root Distinguished Name (DN) entry** for the user who has the unrestricted access to **perform all administration activities** on LDAP, like a root user.

  - **olcRootPW** – LDAP admin password for the above RootDN.

  - create a **db.ldif** file and send the configuration to the LDAP server `ldapmodify -Y EXTERNAL -H ldapi:/// -f db.ldif`

    ```
    dn: olcDatabase={2}hdb,cn=config
    changetype: modify
    replace: olcSuffix
    olcSuffix: dc=A063021,dc=nasa
    
    dn: olcDatabase={2}hdb,cn=config
    changetype: modify
    replace: olcRootDN
    olcRootDN: cn=manager,dc=A063021,dc=nasa # Do not use Syncer, it can bind from any machine
    
    dn: olcDatabase={2}hdb,cn=config
    changetype: modify
    replace: olcRootPW
    olcRootPW: <You keep before>
    ```

- Restrict the monitor access only to ldap root (ldapadm) user not to others. Create a **monitor.ldif** file and send the configuration to the LDAP server `ldapmodify -Y EXTERNAL -H ldapi:/// -f monitor.ldif` (server status monitoring)

  ```
  dn: olcDatabase={1}monitor,cn=config
  changetype: modify
  replace: olcAccess
  olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external, cn=auth" read by dn.base="cn=Syncer,dc=A063021,dc=nasa" read by * none
  // 指定目錄的ACL，即誰有甚麼權限可以存取甚麼
  ```

- Create a **config.ldif** file and send the configuration to the LDAP server `ldapmodify -Y EXTERNAL -H ldapi:/// -f config.ldif` (for cn=Syncer,dc=A063021,dc=nasa to have permission to update schema)

  ```
  dn: olcDatabase={0}config,cn=config
  changetype: modify
  replace: olcAccess
  olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external, cn=auth" manage by dn.base="cn=Syncer,dc=A063021,dc=nasa" manage by * none
  ```

  

### Set up LDAP database (Only master)

- Copy the sample database configuration file to /var/lib/ldap and update the file permissions.

  ```
  cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
  chown ldap:ldap /var/lib/ldap/*
  ```

- Add the cosine and nis LDAP schemas.

  ```
  ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
  ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif 
  ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
  ```

- Generate `base.ldif` file for your domain and build the directory structure `ldapadd -x -D "cn=Syncer,dc=A063021,dc=nasa" -w hahaYouCatchMe -f base.ldif`.

  ```
  dn: dc=A063021,dc=nasa
  dc: A063021
  objectClass: top
  objectClass: domain
  
  dn: cn=Syncer,dc=A063021,dc=nasa
  objectClass: account
  objectClass: posixAccount
  cn: Syncer
  uid: Syncer
  uidNumber: 2999
  gidNumber: 2999
  homeDirectory: /home/Syncer
  userPassword: {SSHA}SwXqgAoIBR38TN7FBNVJh26Pf1zB3Y0M
  
  dn: ou=ldap,dc=A063021,dc=nasa
  objectClass: organizationalUnit
  ou: ldap
  ```


- cluster `cluster.ldif` and then `ldapadd -x -D "cn=Syncer,dc=A063021,dc=nasa" -w hahaYouCatchMe -f cluster.ldif`.

  ```
  dn: cn=master,ou=ldap,dc=A063021,dc=nasa
  objectClass: clusterInfo
  address: 10.113.74.11
  
  dn: cn=slave,ou=ldap,dc=A063021,dc=nasa
  objectClass: clusterInfo
  address: 10.113.74.12
  ```

- Users `users.ldif` and then `ldapadd -x -D "cn=Syncer,dc=A063021,dc=nasa" -w hahaYouCatchMe -f users.ldif`.

  ```
  dn: uid=TA,dc=A063021,dc=nasa
  objectClass: account
  objectClass: posixAccount
  objectClass: shadowAccount
  objectClass: publicKeyLogin
  cn: TA
  uid: TA
  uidNumber: 3000
  gidNumber: 3000
  homeDirectory: /home/TA
  sshPublicKey: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDHW6HKWBol4+vFtDcc2i07PGaoUgbJqjHfEYkbGm5bcebe8oqcsbAEXdNB0TLhIarP2947GYyBfBnX+xorem1MAlJiGntX86GtNft97i5Wg7TxRcvcfWMoIvoW/dDk3C+Na1N2kasKEhQud/4PouvhzcS17KRIqRGM4dLEeFNBCXDyJ/MlpKuZ0ETjipuiF0Jw8IBJeQJ+MLyrlbD/uDO9qqZdcuDx30Kg7jHyd83esfsNR46Bc90gtC+pSYw3Xflfd4Mx522dNZRHnNpM/tcKluZ5dVr/sEAwF4JKd5hAtFq3WPOK9W8UgtC1Nitc7ye16l7a3GlTMeyyX3mGZZgdhp2eHkle4Pa9bbZ4EjtvTF6ek+NEmaCHHeMNzAZtGOOI9Hdn+BZAJb4JosKW1oCZTx/8Gu1kaaa5EMh2ozZJ5h1PG6nIU0Z8vRslurYviXzaURSlhcCFw0MYQ64WePRAMpRqNnr3XgexM1Bu6iUK588DzEcqCGd1a8NWInHk7hU7BfY9dsD/KgP/QCKXPTyHQnUXUaNqKwAhZl+FCzT1jHCQvKKNguS+LdLZls7qy3WZA9iCOwLHZinVBd+SZ5Qc94Cc+lt+IjS2UiwIDajOsy4LPmZcv0V5PE30IRjn6+8aG/WSQuswno42UV2ArtKW61qedzfZPrbS9LpHDM5Nqw== ta@nasa.cs.nctu.edu.tw
  
  dn: uid=A063021,dc=A063021,dc=nasa
  objectClass: account
  objectClass: posixAccount
  objectClass: shadowAccount
  objectClass: publicKeyLogin # add
  cn: A063021
  uid: A063021
  uidNumber: 3001
  gidNumber: 3001
  userPassword: {SSHA}QbRSItqjb0So7YdyKoftz6UDVd2lvCKH # zxcv9841
  homeDirectory: /home/A063021
  sshPublicKey: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC5frH2BUP6F5WWQ4/4AkrS3eFWKGFCpGDj7q8cJwM8ddnRBvSzKSRFf6KMAVAbu9E3/t81Fb6RqILJutKKmPv6q9SugXUem2GI460D1LU1byyX3xtenPzv7CnJoSs+u1k5gETe6MtHRwzEqeoxWedZX+U4b8JkL31Z+6eQYg+0lST1eFL94t27DFVzTPsOaB9k1q8C3vPgPStAZDXaTY5pU/ZSFP45J4pqp/8TzsrmisgBzBXyt5XJLFsRWbqT91u4LiMRnNdwkfFulsadRon8dzNg3egf8ec6L+YSNrf5sWZCXQ2wwHwkchQAMYH7iLQxY298rqOEGuPRnCoXJAjd USER@DESKTOP-FJ8U9C0
  ```

- LDAP search `ldapsearch -x -b 'dc=A063021,dc=nasa'`



## backup and restore

```
slapcat > backup.ldif
systemctl stop slapd
slapadd -c -l backup.ldif
chown -R ldap:ldap /var/lib/ldap
systemctl start slapd
```





### reference

[Centos 7：安裝openldap servers + clients ](https://coodie-h.blogspot.com/2017/09/centos-7openldap.html)

[Step by Step OpenLDAP Server Configuration on CentOS 7 / RHEL 7](https://www.itzgeek.com/how-tos/linux/centos-how-tos/step-step-openldap-server-configuration-centos-7-rhel-7.html)

https://www.linuxbookcenter.com/configuring-ldap-based-authentication-rhel7/

 OpenLDAP 教學 http://benjr.tw/27692



[LDAP基础：7：使用ldapmodify和ldapdelete进行修改或删除](https://blog.csdn.net/liumiaocn/article/details/83991112)



Error

- restart之後config消失 
  - https://serverfault.com/questions/917779/slapd-database-gets-corrupted-on-service-restart
- OpenLDAP common errer
  - https://www.openldap.org/doc/admin24/appendix-common-errors.html



