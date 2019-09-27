# SSH

## Public Key Auth

Do following steps on every machine



`/usr/local/bin/fetchSSHKeysFromLDAP` (chmod 500)

```
#!/bin/sh

ldapsearch -x '(&(objectClass=publicKeyLogin)(uid='"$1"'))' 'sshPublicKey' | \
    sed -n '/^ /{H;d};/sshPublicKey:/x;$g;s/\n *//g;s/sshPublicKey: //gp'
# 手動增加 -b "dc=A063021,dc=nasa" (不建議)
```



`/etc/openldap/ldap.conf`

```
# point to the right LDAP server(s)
BASE	dc=A063021,dc=nasa
URI	    ldap://localhost ldap://10.113.74.12
```



Add the following lines to your `/etc/ssh/sshd_config` file:

```
PubkeyAuthentication yes

AuthorizedKeysCommand /usr/local/bin/fetchSSHKeysFromLDAP
AuthorizedKeysCommandUser root
# AuthorizedKeysCommandRunAs root
```



### Testing

- The output of the script should be identical to the output of your `id_rsa.pub` file

  `/usr/local/bin/fetchSSHKeysFromLDAP A063021`

- log in from a remote computer to the one you want to access:

  `ssh -i ~/.ssh/rsa_id A063021@10.113.74.11 -v` 



## Password

### CentOS

```
yum install -y openldap-clients nss-pam-ldapd

# may skip to follow StartTLS config
authconfig --enableldap --enableldapauth --ldapserver=ldap://10.113.74.11,ldap://10.113.74.12 --ldapbasedn="dc=A063021,dc=nasa" --enablemkhomedir --update
```



### Ubuntu

```
sudo apt-get -y install libnss-ldap libpam-ldap ldap-utils nscd
sudo vim /etc/nsswitch.conf
# Update the below lines shown like below.

passwd:         compat ldap
group:          compat ldap
shadow:         compat ldap

sudo vim /etc/pam.d/common-session
# Add below line in the above file.

session required        pam_mkhomedir.so skel=/etc/skel umask=077

# Restart the nscd service.
sudo service nscd restart
```

#### reconfigure

```
會詢問 LDAP 的設定
sudo dpkg-reconfigure ldap-auth-config

接著用 auth-client-config 設定 LDAP profile：
sudo auth-client-config -t nss -p lac_ldap

最後更新系統 PAM 認證方式：
sudo pam-auth-update
```





Check user exists or not

`getent passwd TA`



### reference

- Key
  - [LDAP-Openssh](http://pig.made-it.com/ldap-openssh.html)
  - [SSH key authentication using LDAP](https://serverfault.com/questions/653792/ssh-key-authentication-using-ldap)

- Client setting
  - [How to Set Up LDAP Authentication with OpenLDAP on CentOS 7](https://hostadvice.com/how-to/how-to-set-up-ldap-authentication-with-openldap-on-centos-7/)
  - https://www.itzgeek.com/how-tos/linux/ubuntu-how-tos/configure-ldap-client-on-ubuntu-16-04-debian-8.html
  - https://blog.gtwang.org/linux/ubuntu-ldap-server/