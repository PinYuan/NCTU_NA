# Build custom schema

## Get your own OID

### Method 1

Run the following commands inPowerShell

```powershell
#--- 
$Prefix="1.2.840.113556.1.8000.2554" 
$GUID=[System.Guid]::NewGuid().ToString() 
$Parts=@() 
$Parts+=[UInt64]::Parse($guid.SubString(0,4),"AllowHexSpecifier") 
$Parts+=[UInt64]::Parse($guid.SubString(4,4),"AllowHexSpecifier") 
$Parts+=[UInt64]::Parse($guid.SubString(9,4),"AllowHexSpecifier") 
$Parts+=[UInt64]::Parse($guid.SubString(14,4),"AllowHexSpecifier") 
$Parts+=[UInt64]::Parse($guid.SubString(19,4),"AllowHexSpecifier") 
$Parts+=[UInt64]::Parse($guid.SubString(24,6),"AllowHexSpecifier") 
$Parts+=[UInt64]::Parse($guid.SubString(30,6),"AllowHexSpecifier") 
$OID=[String]::Format("{0}.{1}.{2}.{3}.{4}.{5}.{6}.{7}",$prefix,$Parts[0],$Parts[1],$Parts[2],$Parts[3],$Parts[4],$Parts[5],$Parts[6]) 
$oid 
#---

#--- output: 1.2.840.113556.1.8000.2554.23998.3901.18320.19951.44236.708794.10266585
```



### Method 2

Apply Private Enterprise Numbers (PEN) on http://pen.iana.org/pen/PenApplication.page

My Private Enterprise Number is: 53859 and my base OID is 1.3.6.1.4.1.53859.



## Define objectClass and attribute

|             | OID base value        |
| ----------- | --------------------- |
| objectClass | 1.3.6.1.4.1.53859.1.1 |
| attribute   | 1.3.6.1.4.1.53859.1.2 |



In ` /root/openldap/schema/` (any path you like but not in /etc/openldap/slapd/)

```
# clusterInfo.schema
# address
attributetype ( 1.3.6.1.4.1.53859.1.2.1
 	NAME 'address' 
 	DESC 'LDAP server IP address'
 	EQUALITY caseIgnoreMatch
    SUBSTR caseIgnoreSubstringsMatch
 	SYNTAX 1.3.6.1.4.1.1466.115.121.1.15{15} )

# clusterinfo
objectclass ( 1.3.6.1.4.1.53859.1.1.1
	NAME 'clusterinfo' SUP top STRUCTURAL
 	DESC 'LDAP cluster info objectclass'
 	MUST ( cn $ address ) 
 )

# opensshLdap.schema
# sshPublicKey
attributetype ( 1.3.6.1.4.1.53859.1.2.2
 	NAME 'sshPublicKey' 
	DESC 'MANDATORY: OpenSSH Public key' 
 	EQUALITY octetStringMatch
 	SYNTAX 1.3.6.1.4.1.1466.115.121.1.40 )
  
# publicKeyLogin 
objectclass ( 1.3.6.1.4.1.53859.1.1.2
	NAME 'publicKeyLogin' SUP top STRUCTURAL
	DESC 'MANDATORY: OpenSSH LPK objectclass'
 	MUST ( sshPublicKey ) )
```



Build a temp folder `/root/openldap/config` and build a file `slapd.conf`

```
include /root/openldap/schema/clusterInfo.schema
include /root/openldap/schema/opensshLdap.schema
```



Run `slaptest -f slapd.conf -F /root/openldap/config/testdir` and it would create a new `cn=config` directory.



Rename

```
mv cn\=\{0\}clusterinfo.ldif clusterinfo.ldif
mv cn\=\{1\}opensshldap.ldif opensshldap.ldif
```



Modify thie files like the following

```
# AUTO-GENERATED FILE - DO NOT EDIT!! Use ldapmodify.
# CRC32 92e2329f
dn: cn=clusterinfo,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: clusterinfo
olcAttributeTypes: {0}( 1.3.6.1.4.1.53859.1.2.1 NAME 'address' DESC 'LDAP se
 rver IP address' EQUALITY caseIgnoreMatch SUBSTR caseIgnoreSubstringsMatch
 SYNTAX 1.3.6.1.4.1.1466.115.121.1.15{15} )
olcObjectClasses: {0}( 1.3.6.1.4.1.53859.1.1.1 NAME 'clusterinfo' DESC 'LDAP
  cluster info objectclass' SUP top AUXILIARY MUST address )
```



Add configuration into LDAP server

`ldapadd -x -D "cn=Syncer,dc=A063021,dc=nasa" -w hahaYouCatchMe -f clusterinfo.ldif`

`ldapadd -x -D "cn=Syncer,dc=A063021,dc=nasa" -w hahaYouCatchMe -f opensshldap.ldif`



At last, restart the OpenLDAP `systemctl restart slapd`



## Testing

create`cluster.ldif` and then `ldapadd -x -D "cn=Syncer,dc=A063021,dc=nasa" -w hahaYouCatchMe -f cluster.ldif`

```
dn: cn=master,ou=ldap,dc=A063021,dc=nasa
objectClass: clusterinfo
address: '10.113.74.11'

dn: cn=slave,ou=ldap,dc=A063021,dc=nasa
objectClass: clusterinfo
address: '10.113.74.12'
```







### reference

[Generating OID using PowerShell (Microsoft Link)](http://fkazi.blogspot.com/2013/04/creating-custom-active-directory_27.html)

- custom schema
  - [LDAP structure](http://www.zytrax.com/books/ldap/ch3/)
  - [OpenLDAP: Create a custom LDAP schema](https://guillaumemaka.com/2013/07/17/openldap-create-a-custom-ldap-schema.html)
  - [How to add a new schema to openLDAP 2.4+](http://stezz.blogspot.com/2012/05/how-to-add-new-schema-to-openldap-24.html)
  - **perfect**  [OpenLDAP using OLC (cn=config)](http://www.zytrax.com/books/ldap/ch6/slapd-config.html#use-schemas)



- update schema but have insufficient-access error
  - https://unix.stackexchange.com/questions/353350/centos-7-ldap-add-insufficient-access-50
  - http://www.mehic.info/2014/05/rootdn-ldap_add-insufficient-access-50/

