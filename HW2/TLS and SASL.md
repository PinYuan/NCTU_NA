# StartTLS and SASL

## StartTLS 

Listen on 389 port (ldap://)

### Server

Create LDAP certificate

- Self-Signed Certificate

  - Generate both certificate and private key

    - `openssl req -new -x509 -nodes -out /etc/openldap/certs/A063021ldap.crt -keyout /etc/openldap/certs/A063021ldap.key -days 1460`

  - Set the owner and group permissions.

    - `chown -R ldap:ldap /etc/openldap/certs/A063021*`

  - Create **certs.ldif** file to configure LDAP to use secure communication using a self-signed certificate and send the configuration to the LDAP server `ldapmodify -Y EXTERNAL  -H ldapi:/// -f certs.ldif`

    ```
    # order is important (不行就換順序)
    dn: cn=config
    changetype: modify
    replace: olcTLSCertificateFile
    olcTLSCertificateFile: /etc/openldap/certs/A063021ldap.crt
    
    dn: cn=config
    changetype: modify
    replace: olcTLSCertificateKeyFile
    olcTLSCertificateKeyFile: /etc/openldap/certs/A063021ldap.key
    ```

  - Verify the configuration `slaptest -u`



configure OpenLDAP to listen over SSL.

```
# vim /etc/sysconfig/slapd

SLAPD_URLS="ldapi:/// ldap:///"
```



Restart the slapd service.

`systemctl restart slapd`



```
# vim  /etc/openldap/ldap.conf
TLS_REQCERT  never # add

# test
ldapsearch  -x  -ZZ
```



### Client

First, use `scp` send the cacert to client. Then, follow these steps depending on OS.

#### CentOS

auto config `/etc/nslcd.conf`

```
authconfig --enableldap --enableldapauth --ldapserver=ldap://10.113.74.11,ldap://10.113.74.12 --ldapbasedn="dc=A063021,dc=nasa" --enablemkhomedir --enableldapstarttls --ldaploadcacert=file:///root/openldap/cacerts/A063021ldap.crt  --update
```



vim` /etc/openldap/ldap.conf `

```
TLS_REQCERT  never # add
```



vim `/etc/nslcd.conf`

```
# The below setting will disable the certificate validation done by clients as we are using a self-signed certificate.

ssl  start_tls
tls_reqcert  never
```



restart nslcd

```
systemctl restart nslcd
```



#### Ubuntu

append cacert into `/etc/ssl/certs/ca-certificates.crt`

```
cat ~/A063021ldap.crt | sudo tee -a /etc/ssl/certs/ca-certificates.crt
```



modify the cacert path if you set a different path

```
# /etc/ldap/ldap.condf
# TLS certificates (needed for GnuTLS)
TLS_CACERT      /etc/ssl/certs/ca-certificates.crt
```



 vim `/etc/openldap/ldap.conf `

```
TLS_REQCERT  never # add
```



### Test STARTTLS

- server
  - `ldapwhoami -H ldap:// -x -ZZ`
- client
  - `ldapwhoami -H ldap://10.113.74.11 -x -ZZ`



## SASL

To modify or set ldap passwords using slapd.conf (rootpw) or ldapmodify/ldapadd (userPassword) one can use slappasswd utility. To change password using slappasswd use following steps:

1. Run '`slappasswd`' command and enter the desired password twice.
2. Copy the output password usually protected using {SSHA} including the {SSHA} tag and paste it in '`slapd.conf`' file or in ldif file to be used with '`ldapadd`' or '`ldapmodify`'
3. Restart slapd in case root password was changed or run '`ldapmodify`' or '`ldapadd`' in case of userPassword being modified in ldif format.

The advantage of this approach over storing plain-text password is that even if password-database is lost, it would take sometime for users to recover their passwords. Use of cleartext passwords is not recommended. Note that ldapsearch output will display passwords in base64 encoded format. Hence use base64_decode function to get the stored password in format usable by other programs.





### reference

- SSL
  - [Configure OpenLDAP with SSL on CentOS 7 / RHEL 7](https://www.itzgeek.com/how-tos/linux/centos-how-tos/configure-openldap-with-ssl-on-centos-7-rhel-7.html)
  - http://phorum.study-area.org/index.php?topic=68194.0
  - https://www.howtoing.com/how-to-encrypt-openldap-connections-using-starttls
  - https://help.ubuntu.com/community/SecuringOpenLDAPConnections

- SASL
  - https://www.sbarjatiya.com/notes_wiki/index.php/Securing_openLDAP_SASL_authentication

