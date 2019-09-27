# DNS

## Install

```
yum install -y bind bind-utils
# bind-utils installs nslookup, dig and host
```



## Master

### Main setting

vim `/etc/named.conf`

```
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//
// See the BIND Administrator's Reference Manual (ARM) for details about the
// configuration located in /usr/share/doc/bind-{version}/Bv9ARM.html

options {
        listen-on port 53 { 127.0.0.1; 10.113.74.1 }; # modify
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { localhost; 10.113.0.0/16; }; # modify
        allow-transfer  { 127.0.0.1; 10.113.74.2; 10.113.0.0/21; }; # add
        notify yes;

        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        recursion yes;
        allow-recursion { localhost; 10.113.74.0/24; }; # add

        dnssec-enable yes;
        dnssec-validation yes;

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.iscdlv.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

view "scope1" {
    match-clients {10.113.2.0/24; 10.113.3.0/24;};
	zone "." IN {
        type hint;
        file "named.ca";
	};
	zone "nasa." IN {
    	type slave;
    	file "slaves/zone-nasa";
        masters { 10.113.0.254; };
	};
    zone "A063021.nasa" IN {
        type master;
        file "scope1/zone-A063021.nasa";
    }; 
    zone "74.113.10.in-addr.arpa" IN {
        type master;
        file "scope1/zone-10.113.74.0";
    };
}

view "scope2" {
    match-clients {10.113.1.0/24; 10.113.4.0/24;};
	zone "." IN {
        type hint;
        file "named.ca";
	};
	zone "nasa." IN {
    	type slave;
    	file "slaves/zone-nasa";
        masters { 10.113.0.254; };
	};
    zone "A063021.nasa" IN {
        type master;
        file "scope2/zone-A063021.nasa";
    }; 
    zone "74.113.10.in-addr.arpa" IN {
        type master;
        file "scope2/zone-10.113.74.0";
    };
}

view "scope3" {
    match-clients {any;};
	zone "." IN {
        type hint;
        file "named.ca";
	};
	zone "nasa." IN {
    	type slave;
    	file "slaves/zone-nasa";
        masters { 10.113.0.254; };
	};
    zone "A063021.nasa" IN {
        type master;
        file "scope3/zone-A063021.nasa";
    }; 
    zone "74.113.10.in-addr.arpa" IN {
        type master;
        file "scope3/zone-10.113.74.0
    };
}

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

### Zone file

#### Scope1 Forwarding

vim `/var/named/scope1/zone-A063021.nasa`

```
$TTL    86400
$ORIGIN  A063021.nasa.
@     IN SOA    ns1.A063021.nasa. webmaster.A063021.nasa. ( 2019050822 1D 30M 1W 2H )

      IN NS     ns1.A063021.nasa.
      IN NS     ns2.A063021.nasa.
ns1  IN A      10.113.74.1
ns2  IN A      10.113.74.2

router  IN A    10.113.74.254

ldap1   IN A    10.113.74.11
ldap2   IN A    10.113.74.12

nasa IN CNAME nasa.cs.nctu.edu.tw.
friend IN CNAME ns2

view    IN A    10.113.235.131
```

#### Scope1 Reversing

vim `/var/named/scope1/zone-10.113.74.0`

```
$TTL     86400
$ORIGIN  74.113.10.in-addr.arpa.
@  IN SOA  ns1.A063021.nasa. webmaster.A063021.nasa. ( 2019050822 1D 30M 1W 2H )

   IN NS   ns1.A063021.nasa.
   IN NS   ns2.A063021.nasa.
1  IN PTR  ns1.A063021.nasa.
2  IN PTR  ns2.A063021.nasa.

254  IN PTR  router.A063021.nasa.

11  IN PTR  ldap1.A063021.nasa.
12  IN PTR  ldap2.A063021.nasa.
```



#### Scope2 Forwarding

vim `/var/named/scope2/zone-A063021.nasa`

```
$TTL    86400
$ORIGIN  A063021.nasa.
@     IN SOA    ns1.A063021.nasa. webmaster.A063021.nasa. ( 2019050822 1D 30M 1W 2H )

      IN NS     ns1.A063021.nasa.
      IN NS     ns2.A063021.nasa.
ns1  IN A      10.113.74.1
ns2  IN A      10.113.74.2

router  IN A    10.113.74.254

ldap1   IN A    10.113.74.11
ldap2   IN A    10.113.74.12

nasa IN CNAME nasa.cs.nctu.edu.tw.
friend IN CNAME ns2

view    IN A    10.113.235.151
```

#### Scope2 Reversing

same with scope1 -> vim `/var/named/scope2/zone-10.113.74.0`



#### Scope3 Forwarding

vim `/var/named/scope3/zone-A063021.nasa`

```
$TTL    86400
$ORIGIN  A063021.nasa.
@     IN SOA    ns1.A063021.nasa. webmaster.A063021.nasa. ( 2019050822 1D 30M 1W 2H )

      IN NS     ns1.A063021.nasa.
      IN NS     ns2.A063021.nasa.
ns1  IN A      10.113.74.1
ns2  IN A      10.113.74.2

router  IN A    10.113.74.254

ldap1   IN A    10.113.74.11
ldap2   IN A    10.113.74.12

nasa IN CNAME nasa.cs.nctu.edu.tw.
friend IN CNAME ns2

view    IN A    10.113.74.87
```

#### Scope3 Reversing

same with scope1 -> vim `/var/named/scope3/zone-10.113.74.0`



## Check syntex

```
named-checkconf /etc/named.conf
named-checkzone A063021.nasa /var/named/xxx/zone-A063021.nasa
named-checkzone 74.113.10.in-addr.arpa /var/named/xxx/zone-10.113.74.0
```



## Restart service

```
systemctl restart named
systemctl enable named # enable after boot
```



## Firewall

```
firewall-cmd --permanent --zone=public --add-port=53/tcp
firewall-cmd --permanent --zone=public --add-port=53/udp
firewall-cmd --reload 
```



## Slave

1. vim `/etc/named.conf`

```
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//
// See the BIND Administrator's Reference Manual (ARM) for details about the
// configuration located in /usr/share/doc/bind-{version}/Bv9ARM.html

options {
        listen-on port 53 { 127.0.0.1; 10.113.74.2; }; # modify
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { localhost; 10.113.0.0/16; }; # modify

        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        recursion yes;
        allow-recursion { localhost; 10.113.74.0/24; }; # add

        dnssec-enable yes;
        dnssec-validation yes;

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.iscdlv.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

zone "nasa." IN {
    	type slave;
    	file "slaves/zone-nasa";
        masters { 10.113.0.254; };
};

zone "A063021.nasa" IN {
    type    slave;
    file    "slaves/zone-A063021.nasa";
    masters { 10.113.74.1; };
}; 

zone "74.113.10.in-addr.arpa" IN {
    type    slave;
    file    "slaves/zone-10.113.74.0";
    masters { 10.113.74.1; };
}; 

zone "sec.A063021.nasa" IN {
    type    slave;
    file    "slaves/zone-sec.A063021.nasa";
    masters { 10.113.74.1; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```



2. check whether config file has error

3. open firewall

4. restart service



## SSHFP

Do on each machine`ssh-keygen -r <hostname>`, then add the output into zone file (choose the algo. you want to restrict).

```
# Example
[root@ns1 ~]# ssh-keygen -r ns1.A063021.nasa
router.A063021.nasa IN SSHFP 1 1 68e7a5f624729071ad7aa7e17f6a5b62204ce24a
router.A063021.nasa IN SSHFP 1 2 1f50ef6269ef4d11e279a4bcd48f76f8e97b1dc5032a723bf8fedc8fa844c332
router.A063021.nasa IN SSHFP 3 1 4c4358345019fda364febd25a555176803d7325b
router.A063021.nasa IN SSHFP 3 2 67f5a6fa6a1d194a2d0ade526bb74ed5c70c88d683dc10a00cec644a94c3809e
router.A063021.nasa IN SSHFP 4 1 7dcd70d443c215b399f893c8a49efa2e785ba328
router.A063021.nasa IN SSHFP 4 2 16257806f1486c437459d9fb0845938166b0fc688c3e49e3bb1d19d3ed4df221
```



### Testing

` ssh -o "VerifyHostKeyDNS=yes" hostname` shows "Matching host key fingerprint found in DNS".

```
The authenticity of host 'ns2.a063021.nasa (10.113.74.2)' can't be established.
ECDSA key fingerprint is SHA256:M+iiwG6kCG1buI09/kkziCnoW5+A/R6v8jY9Z3vn4qI.
Matching host key fingerprint found in DNS.
ECDSA key fingerprint is MD5:7e:d7:d0:42:7c:52:1e:c8:0a:c3:b2:d8:a6:d3:f3:19.
Matching host key fingerprint found in DNS.
Are you sure you want to continue connecting (yes/no)?
```



## Delegate subdomain to master (myself)

### Domain Name-Server named.conf may not needed

### Domain Name-Server Zone File

```
$TTL    86400
$ORIGIN  A063021.nasa.
@       IN SOA  ns1.A063021.nasa. webmaster.A063021.nasa. ( 2019050822 1D 30M 1W 2H )

        IN NS   ns1.A063021.nasa.
        IN NS   ns2.A063021.nasa.
ns1     IN A    10.113.74.1
ns2     IN A    10.113.74.2

router  IN A    10.113.74.254

ldap1   IN A    10.113.74.11
ldap2   IN A    10.113.74.12

nasa    IN CNAME        nasa.cs.nctu.edu.tw.
friend  IN CNAME        ns2

view    IN A    10.113.235.131

# SSHFP ...
# DS ...

$ORIGIN sec.A063021.nasa.
@                       IN NS   ns1.A063021.nasa.
```



### Sub-domain named.conf

```
zone "sec.A063021.nasa" in {
	type master;
	file "sec.A063021.nasa";
}
```



### Sub-domain Zone Files

```
$TTL    86400
$ORIGIN sec.A063021.nasa.
@              IN     SOA   id.sec.A063021.nasa. webmaster.sec.A063021.nasa. ( 20190501520 1D 30M 1W 2H )
               IN     NS    ns1.A063021.nasa.
id	           IN     TXT   A063021
```





## DNSSEC

DNSKEY

```
mkdir /var/named/keys
dnssec-keygen -a RSASHA512 -b 2048 -3 -f KSK -r /dev/urandom sec.A063021.nasa
dnssec-keygen -a RSASHA512 -b 2048 -3 -r /dev/urandom sec.A063021.nasa
chown named:named *
```

add the public keys which contain the DNSKEY record to the zone file

```
$INCLUDE "/var/named/keys/Ksec.a063021.nasa.+010+35945.key"
$INCLUDE "/var/named/keys/Ksec.a063021.nasa.+010+47241.key"
```



Sign the zone with the dnssec-signzone command.

```
dnssec-signzone -3 61 -A -N INCREMENT -o sec.A063021.nasa -t -K /var/named/keys zone-sec.A063021.nasa
```



Add new signed zone file to named.conf



Create chain of trust, copy DS record (dsset.sec.A063021.nasa.) to parent zone file.

```
sec.A063021.nasa.       IN DS 47241 10 1 3B7D142B067B3E493847600A18BFE5F5420E0B5D
sec.A063021.nasa.       IN DS 47241 10 2 FD9D2D572D50A0E1138852DB5AF7472F3FE5DF9CDE56D01C86A6B7F9 6C5404DE
```



## Key word

- zone tranfer
  - It is the way for DNS slave server to get replication from its master. So for master, it needs to control the zone transfer list to prevent anyone can access DN list.
  - if without para `notify`, slave will be updated on refresh time
  - Master: limit range, Slave: disable allow-transfer

- SSHFP 

  - SSHFP RR records are DNS records that contain fingerprints for public keys used for SSH. They're mostly used with DNSSEC enabled domains. When a SSH client connects to a server it checks the corresponding SSHFP record. If the records fingerprint matches the servers, the server is legit and it's safe to connect.

- DNSSEC
  - http://ichiro-note.blogspot.com/2012/07/dnssec.html

  - https://www.lijyyh.com/2012/07/dnssec-introduction-to-dnssec.html

    

### Reference

- set master
  - http://blog.itist.tw/2016/02/building-dns-server-with-bind-on-centos-7.html
- set slave
  - http://blog.itist.tw/2016/02/setup-slave-dns-server-with-bind-on-centos-7.html

http://dns-learning.twnic.net.tw/bind/intro3.html

- zone transfer
  - https://net.nthu.edu.tw/2009/dns:bind_zone_transfer
- SSHFP
  - https://unix.stackexchange.com/questions/121880/how-do-i-generate-sshfp-records
  - https://datahunter.org/sshfp 
  - https://ayesh.me/sshfp-verification
  - https://blog.gslin.org/archives/2018/08/07/8435/ssh-%E7%9A%84-sshfp-record/
- DNSSEC
  - https://www.nanog.org/sites/default/files/Deccio_Hands-On_Dnssec.pdf
  - https://blog.webernetz.net/dnssec-with-nsec3/
  - https://www.digitalocean.com/community/tutorials/how-to-setup-dnssec-on-an-authoritative-bind-dns-server--2 
  - http://www.tcrc.edu.tw/files/44/4.pdf 目前照這個