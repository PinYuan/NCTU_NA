# ACL

## Data

1. Everyone can read all data except userPassword
2. Authenticated user can write their own userPassword
3. "cn=Syncer" can read all data



## Bind

Only slave server can bind to  DN "cn=Syncer" 



Create a **acl.ldif** file and send the configuration to the LDAP server `ldapmodify -Y EXTERNAL -H ldapi:/// -f acl.ldif` (Do on all LDAP servers, note to update peername)

```
dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to dn.base="cn=Syncer,dc=A063021,dc=nasa"
        by peername.ip=10.113.74.11 manage
        by peername.ip=10.113.74.12 auth
        by users none
        by * none
-
add: olcAccess
olcAccess: {1}to attrs=userPassword
        by self write
        by dn.base="cn=Syncer,dc=A063021,dc=nasa" read
        by anonymous auth
        by * none
-
add: olcAccess
olcAccess: {2}to * 
        by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external, cn=auth" manage 
        by dn.base="cn=Syncer,dc=A063021,dc=nasa" manage 
        by * read
```



### reference

https://www.openldap.org/doc/admin24/access-control.html (8.4.3)

https://www.openldap.org/lists/openldap-software/200711/msg00342.html