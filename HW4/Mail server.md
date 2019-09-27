# Mail server

## Concept

- [SMTP/POP](http://learn-web-hosting-domain-name.mygreatname.com/how-mail-server-works/how-email-send-receive-works.html)

  

## Postfix (SMTP)

### Install

` yum -y install postfix`

### Config 

`vim /etc/postfix/main.cf`

```bash
# line 75: uncomment and specify hostname
myhostname = mail.A063021.nasa
# line 83: uncomment and specify domain name
mydomain = A063021.nasa
# line 98: uncomment 設定header的mail from
myorigin = $myhostname
# line 113: change 收信源
inet_interfaces = all
# line 164: add 用來收件的主機名稱
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
# line 264: uncomment and specify your local network 可寄信的網域  Relay domain
mynetworks = 127.0.0.0/8, 10.113.74.0/24
# line 419: uncomment
    home_mailbox = Maildir/
# line 574: add 變更 smtp 連線招呼語 [SKIP]
smtpd_banner = $myhostname ESMTP
# add follows to the end  [size are skipped]
# limit an email size for 10M
message_size_limit = 10485760
# limit a mailbox for 1G
mailbox_size_limit = 1073741824
# for SMTP-Auth
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain = $myhostname
smtpd_recipient_restrictions = permit_mynetworks,permit_auth_destination,permit_sasl_authenticated,reject_unauth_destination
```

#### Check config file

`postfix check`

### Restart service

`systemctl restart postfix`

`systemctl enable postfix`

### Firewall

`firewall-cmd --add-service=smtp --permanent`
`firewall-cmd --reload`

### Reference

- http://linux.vbird.org/linux_server/0390postfix.php

- (Postfix+Dovecot+SSL)https://codertw.com/%E4%BC%BA%E6%9C%8D%E5%99%A8/379948/



## Dovecot (POP/IMAP)

### Install

`yum -y install dovecot`

### Config

`vim /etc/dovecot/dovecot.conf`

```
protocols = imap pop3 lmtp
listen = *, ::
```



`vim /etc/dovecot/conf.d/10-auth.conf`

```
disable_plaintext_auth = no
auth_mechanisms = plain login
```



`vim /etc/dovecot/conf.d/10-mail.conf`

```
mail_location = maildir:~/Maildir
```



`vim /etc/dovecot/conf.d/10-master.conf`

```bash
#unix_listener auth-userdb {
     #mode = 0600
     #user =
     #group =
   #}
# Postfix smtp-auth
unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
}
```

### Restart service

` systemctl start dovecot`

` systemctl enable dovecot`

### Firewall

`firewall-cmd --add-port={110/tcp,143/tcp} --permanent`
`firewall-cmd --reload`

### Reference

- https://support.rackspace.com/how-to/dovecot-installation-and-configuration-on-centos/

- (Problem writing maildir) https://unix.stackexchange.com/questions/176732/acl-mac-permissions-for-dovecot-and-postfix-in-centos-7/176734

  - reboot after setting

- > 



## DNS MX

`vim /var/named/scope3/zone-A063021.nasa`

```
@	IN	MX	5	mail
mail	IN A	10.113.74.7
```



## STARTTLS

### Create self-signed certificate

```bash
mkdir /etc/ssl/private
chmod 700 /etc/ssl/private

# create key and cert
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/hw4.key -out /etc/ssl/certs/hw4.crt
```

###Configure Postfix

`vim /etc/postfix/main.cf`

```bash
# add to the end (replace certificates to your own one
smtpd_use_tls = yes
smtp_tls_mandatory_protocols = !SSLv2, !SSLv3
smtpd_tls_mandatory_protocols = !SSLv2, !SSLv3
smtpd_tls_cert_file = /etc/ssl/certs/hw4.crt
smtpd_tls_key_file = /etc/ssl/private/hw4.key
smtpd_tls_session_cache_database = btree:/etc/postfix/smtpd_scache
```



`vim /etc/postfix/master.cf`

```bash
# line 16,17,19: uncomment
submission inet n       -       n       -       -       smtpd
  -o syslog_name=postfix/submission
#  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes

# line 26-28: uncomment
smtps     inet  n       -       n       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
```

###Configure Dovecot

`vim /etc/dovecot/conf.d/10-ssl.conf`

```bash
# line 8: change
ssl = yes
# line 14,15: specify sertificates (replace to your own one)
ssl_cert = </etc/ssl/certs/hw4.crt
ssl_key = </etc/ssl/private/hw4.key
# line 51: uncomment and add
ssl_protocols = !SSLv2 !SSLv3
```

### Restart service

`systemctl restart postfix dovecot`

### Firewall

allow SMTP-Submission/SMTPS/POP3S/IMAPS services. SMTP-Submission uses 587/TCP(used STARTTLS), SMTPS uses 465/TCP, POP3S uses 995/TCP, IMAPS uses 993/TCP                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         

`firewall-cmd --add-service={smtp-submission,smtps,pop3s,imaps} --permanent `

`firewall-cmd --reload`

### Testing

Test using imap port and STARTTLS command (works also with imap port):

`openssl s_client -connect imap.example.com:143 -starttls imap`

### Reference

- https://www.digitalocean.com/community/tutorials/how-to-create-an-ssl-certificate-on-apache-for-centos-7

- https://www.server-world.info/en/note?os=CentOS_7&p=mail&f=4



## SPF

### Add SPF records to DNS

vim DNS zone file

```
@	IN TXT "v=spf1 +mx -all"
```

### Add SPF policy to Postfix

Install `yum install -y pypolicyd-spf`



`vim /etc/postfix/master.cf`

```bash
# add the following entry at the end:
policyd-spf  unix  -       n       n       -       0       spawn
    user=policyd-spf argv=/usr/bin/policyd-spf
```



`vim /etc/postfix/main.cf`

```bash
smtpd_recipient_restrictions =
    ...
    reject_unauth_destination,
    check_policy_service unix:private/policyd-spf,
    ...
    
# increase the Postfix policy agent timeout, which will prevent Postfix from aborting the agent if transactions run a bit slowly
policyd-spf_time_limit = 3600
```



Restart postfix `systemctl restart postfix`

### Reference

- https://www.opencli.com/linux/bind-dns-create-spf-record
- https://www.linode.com/docs/email/postfix/configure-spf-and-dkim-in-postfix-on-debian-8/
- https://medium.com/@ri7nz/setup-postfix-policyd-spf-centos-7-6957e8b4de78
- https://zurgl.com/how-to-configure-spf-in-postfix/



## Add LDAP and Users

### Enable LDAP password login

```bash
yum install -y openldap-clients nss-pam-ldapd

# may skip to follow StartTLS config
authconfig --enableldap --enableldapauth --ldapserver=ldap://10.113.74.11,ldap://10.113.74.12 --ldapbasedn="dc=A063021,dc=nasa" --enablemkhomedir --update
```



## LDAP user auth

###Postfix integrate with LDAP

`postconf -m | grep ldap`



Testing

```
testsaslauthd -s smtp -u A063021 -p <password>
0: OK "Success.
```



###Dovecot integrate with LDAP

`vim /etc/dovecot/conf.d/10-auth.conf`

```
!include auth-system.conf.ext
!include auth-ldap.conf.ext
```



check `auth-system.conf.ext`

```
passdb {
  driver = pam
}
 
userdb {
  driver = passwd
}
```



check `auth-ldap.conf.ext`

```
passdb {
  driver = ldap
  args = /etc/dovecot/dovecot-ldap.conf
}
 
userdb {
  driver = ldap
  args = /etc/dovecot/dovecot-ldap.conf
}
```



`vim /etc/dovecot/dovecot-ldap.conf`

```
hosts = 10.113.74.11:389
dn = cn=Syncer,dc=A063021,dc=nasa
dnpass = hahaYouCatchMe
base = dc=A063021,dc=nasa
```

### Reference

- https://shazi.info/centos-6-5-%EF%BC%8Dldap-%E6%95%B4%E5%90%88%E9%A9%97%E8%AD%89-postfix-dovecot/



## DKIM

Set EPEL Repository using below rpm command

` rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm`



Install OpenDKIM Package using yum

`yum install -y opendkim`



Create keys

```
mkdir /etc/opendkim/keys/A063021.nasa
opendkim-genkey -D /etc/opendkim/keys/A063021.nasa/ -d A063021.nasa -s default
chown -R opendkim: /etc/opendkim/keys/A063021.nasa
mv /etc/opendkim/keys/A063021.nasa/default.private /etc/opendkim/keys/A063021.nasa/default
```



`vim /etc/opendkim.conf`

```
# modify corresponding field
Mode                    sv
Canonicalization        relaxed/simple
Domain                  A063021.nasa
# KeyFile...
KeyTable				refile:/etc/opendkim/KeyTable
SigningTable			refile:/etc/opendkim/SigningTable
ExternalIgnoreList		refile:/etc/opendkim/TrustedHosts
InternalHosts			refile:/etc/opendkim/TrustedHosts
```



`vim /etc/opendkim/KeyTable`

```
default._domainkey.A063021.nasa A063021.nasa:default:/etc/opendkim/keys/A063021.nasa/default
```



`vim /etc/opendkim/SigningTable`

all the users on domain are allowed to sign the emails.

```
*@A063021.nasa default._domainkey.A063021.nasa
```



`vim /etc/opendkim/TrustedHosts`

Server’s FQDN and domain name

```
127.0.0.1
mail.A063021.nasa
A063021.nasa
```



`vim /etc/postfix/main.cf`

```
# append at the end
smtpd_milters = inet:127.0.0.1:8891
non_smtpd_milters = $smtpd_milters
milter_default_action = accept
```



add DKIM record (/etc/opendkim/keys/A063021.nasa/default.txt) to DNS

```
default._domainkey      IN      TXT     ( "v=DKIM1; k=rsa; "
          "p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDh1tY4FI41sVv5+UDGIFL3j6l/SEe4ijMrXnMbm6hQcVMwj+ruBOzaPY0G/9yj45bLUaDdCAB2Nimj1l8PNy5z3a27y+WF1w5GQZmem/6aH8b5xoOwkavz1SzHvpMzLjN2BJSlEUj5fCOo4L2MTDKm3gPO6ceKp+vSIWMi/huzxwIDAQAB" )  ; ----- DKIM key default for A063021.nasa
```



Restart service 

```
systemctl start opendkim
systemctl enable opendkim
systemctl restart postfix
```



### Testing

```
mail from: pinyuan@A063021.nasa
rcpt to: pinyuan615@gmail.com
data
subject: testing 587
test email
.
```



![sendmail-with-telnet](C:\Users\USER\Desktop\NCTU_NA\HW4\sendmail-with-telnet.jpg)



### Reference

- https://www.itread01.com/content/1542104586.html
- https://www.linuxtechi.com/configure-domainkeys-with-postfix-on-centos-7/



## DMARC

Add DMARC record to DNS zone file

```
_dmarc	TXT	"v=DMARC1;p=reject"	
```



Install opendmarc `yum install -y opendmarc`



`vim /etc/opendmarc.conf`

```
AuthservID mail.A063021.nasa
```



`vim /etc/postfix/main.cf`

```
smtpd_milters = unix:private/opendkim unix:private/opendmarc
non_smtpd_milters = unix:private/opendkim unix:private/opendmarc
# or
smtpd_milters           = inet:127.0.0.1:8891, inet:127.0.0.1:8893
non_smtpd_milters       = $smtpd_milters
milter_default_action   = accept
```



Restart service

```
systemctl start opendmarc
systemctl enable opendmarc
```



### Reference

- (spf-dkim-dmarc-postfix) https://petermolnar.net/howto-spf-dkim-dmarc-postfix/
- https://www.stevejenkins.com/blog/2015/03/installing-opendmarc-rpm-via-yum-with-postfix-or-sendmail-for-rhel-centos-fedora/



## Greylisting

Install postgrey ` yum -y install postgrey `



`vim /etc/sysconfig/postgrey`

```
POSTGREY_OPTS="--delay=30"
```



`vim /etc/postfix/main.cf`

```
smtpd_recipient_restrictions = permit_mynetworks,
permit_sasl_authenticated,
reject_unauth_destination,
check_policy_service unix:postgrey/socket,
check_policy_service unix:private/spfpolicy,
permit
```



Restart service

```
systemctl start postgrey
systemctl enable postgrey
systemctl restart postfix
```

### Reference

- https://www2.linuxacademy.com/howtoguides/18200-using-greylisting-with-postfix-in-rhel-7-and-centos-7/
- https://anderson1029.pixnet.net/blog/post/24429807-%E9%98%B2%E6%B2%BB%E5%9E%83%E5%9C%BE%E4%BF%A1:-%E5%9C%A8-sendmail%E3%80%81postfix-%E5%8A%A0%E5%85%A5%E7%81%B0%E5%90%8D%E5%96%AE%E9%81%8E



## Add LDAP user

### Normal user

- password: wDmENYvXbmldleRLjFVfLhu5/k99msQ8FrIdL2NFNXE=

  - slappasswd -h {SSHA} -s wDmENYvXbmldleRLjFVfLhu5/k99msQ8FrIdL2NFNXE=
  - Output - {SSHA}ggmnOufVl59HIlJ0KWDzmmXD09sH3Rma

- Users `TA2user.ldif` and then `ldapadd -x -D "cn=Syncer,dc=A063021,dc=nasa" -w hahaYouCatchMe -f TA2user.ldif`.

  ```
  dn: uid=TA2,dc=A063021,dc=nasa
  objectClass: account
  objectClass: posixAccount
  objectClass: shadowAccount
  objectClass: publicKeyLogin # add
  cn: TA2
  uid: TA2
  uidNumber: 3002
  gidNumber: 3002
  userPassword: {SSHA}ggmnOufVl59HIlJ0KWDzmmXD09sH3Rma
  homeDirectory: /home/TA2
  ```

- LDAP search `ldapsearch -x -b 'dc=A063021,dc=nasa'`



## Virtual alias

`vim /etc/postfix/main.cf`

```
# add in the end
virtual_alias_maps = regexp:/etc/postfix/virtual
```



`vim /etc/postfix/virtual`

```
/TA3@(\S+)/ TA@$1
/(\S+)\|(\S+)@(\S+)/ $2@$3
```



make postfix read the file `postmap /etc/postfix/virtual`

### Reference

- https://www.electrictoolbox.com/update-postfix-virtual-alias-map/



## Sender rewrite

`vim /etc/postfix/main.cf`

```
sender_canonical_maps = regexp:/etc/postfix/sender_canonical
```



`vim /etc/postfix/sender_canonical`

```
/^(.*@)mail.A063021.nasa$/     ${1}A063021.nasa
```

### Reference

- https://www.linuxquestions.org/questions/linux-server-73/rewrite-sender-address-in-postfix-852693/



## Clamav + Amavisd

### **Clamav**

Install

```
yum --enablerepo=epel -y install clamav clamav-update
sed -i -e "s/^Example/#Example/" /etc/freshclam.conf
```



Update pattern files `freshclam`



Try to scan

```
clamscan --infected --remove --recursive /home
```



Create a trial virus file

`vim eicar.com`

```
X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*
```

```
clamscan --infected --remove --recursive .
```



### **Clamav server + Amavisd**

Install

```
yum --enablerepo=epel -y install amavisd-new clamav-scanner clamav-scanner-systemd
```



If SELinux is enabled, change policy like follows.

`setsebool -P antivirus_can_scan_system on `



Configure Amavisd `vim /etc/amavisd/amavisd.conf`

```
# line 20: change to own domain name
$mydomain = 'A063021.nasa';

# line 152: change to the own hostname
$myhostname = 'mail.A063021.nasa';

# line 154: uncomment
$notify_method = 'smtp:[127.0.0.1]:10025';
$forward_method = 'smtp:[127.0.0.1]:10025';

# line 383: make sure settings are like folows
['ClamAV-clamd',
    \&ask_daemon, ["CONTSCAN {}\n", "/var/run/clamd.amavisd/clamd.sock"],
    qr/\bOK$/m, qr/\bFOUND$/m,
    qr/^.*?: (?!Infected Archive)(.*) FOUND$/m ],
    
# for subject add tag
$sa_spam_subject_tag = '***SPAM***';
$subject_tag_maps_by_ccat{+CC_VIRUS} = [ '***SPAM*** ' ];
$final_spam_destiny=D_PASS;
$final_virus_destiny=D_PASS;
```



Start service

```
systemctl start clamd@amavisd amavisd
systemctl enable clamd@amavisd amavisd
```



Configure postfix 

`vim /etc/postfix/main.cf`

```
# add to the end
content_filter=smtp-amavis:[127.0.0.1]:10024
```

`vim /etc/postfix/master.cf`

```
# add to the end
smtp-amavis unix -    -    n    -    2 smtp
    -o smtp_data_done_timeout=1200
    -o smtp_send_xforward_command=yes
    -o disable_dns_lookups=yes
127.0.0.1:10025 inet n    -    n    -    - smtpd
    -o content_filter=
    -o local_recipient_maps=
    -o relay_recipient_maps=
    -o smtpd_restriction_classes=
    -o smtpd_client_restrictions=
    -o smtpd_helo_restrictions=
    -o smtpd_sender_restrictions=
    -o smtpd_recipient_restrictions=permit_mynetworks,reject
    -o mynetworks=127.0.0.0/8
    -o strict_rfc821_envelopes=yes
    -o smtpd_error_sleep_time=0
    -o smtpd_soft_error_limit=1001
    -o smtpd_hard_error_limit=1000
```

Restart service `systemctl restart postfix`

### Reference

- https://www.server-world.info/en/note?os=CentOS_7&p=clamav
- http://docs.trendmicro.com/all/smb/wfbs-services/Server/Dell/v3.7/zh-TW/docs/WebHelp/_WFBS-SVC/C03-InstallingAgents/TestVirusEICAR.htm
- https://www.server-world.info/en/note?os=CentOS_7&p=mail&f=6
- https://serverfault.com/questions/613997/amavis-spamassassin-setup-how-to-disable-quarantine-and-get-default-spamassas



## Header_checks

`vim /etc/postfix/main.cf`

```
header_checks = regexp:/etc/postfix/header_checks
# body_checks = regexp:/etc/postfix/body_checks
```



`vim /etc/postfix/header_checks`

```
/^Subject:.*小熊維尼/ REJECT
```



make db `postmap /etc/postfix/header_checks`



restart service `systemctl restart postfix`

###  Reference

- https://blog.mikuru.tw/archives/36
- https://kafeiou.pw/2018/06/05/689/



## Telnet

SMTP https://ssorc.tw/4499

IMAP https://ssorc.tw/3196