# How to build a slave LDAP server

## Master

Load syncprov module `syncprov_mod.ldif `

```
dn: cn=module{0},cn=config
objectClass: olcModuleList
cn: module
olcModulePath: /usr/lib64/openldap
olcModuleLoad: syncprov.la
```

`ldapadd -Y EXTERNAL -H ldapi:/// -f syncprov_mod.ldif`



Enable syncprov module `syncprov_hdb.ldif`

```
dn: olcOverlay=syncprov,olcDatabase={2}hdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcSyncProvConfig
olcOverlay: syncprov
olcSpSessionLog: 100
```

`ldapadd -Y EXTERNAL -H ldapi:/// -f syncprov_hdb.ldif`



Enable syncprov module `syncprov_config.ldif` (for schema replication)

```
dn: olcOverlay=syncprov,olcDatabase={0}config,cn=config
objectClass: olcOverlayConfig
objectClass: olcSyncProvConfig
olcOverlay: syncprov
olcSpSessionLog: 100
```

`ldapadd -Y EXTERNAL -H ldapi:/// -f syncprov_config.ldif`



## Slave

`enable_sync_consumer.ldif `

```
dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcSyncrepl
olcSyncrepl: rid=000 
  provider=ldap://10.113.74.11:389
  type=refreshonly
  interval=00:00:01:00
  retry="5 5 300 +" 
  searchbase="dc=A063021,dc=nasa"
  attrs="*,+"
  bindmethod=simple
  binddn="cn=Syncer,dc=A063021,dc=nasa"
  credentials=hahaYouCatchMe
```

`ldapmodify -Y EXTERNAL -H ldapi:/// -f enable_sync_consumer.ldif`

### Arguments detail

- [interval=dd:hh:mm:ss]
- retry [retry=[<retry interval> <# of retries>]+]
  - retry="60 10 300 3" lets the consumer retry every 60 seconds for the first 10 times and then retry every 300 seconds for the next three times before stop retrying. + in <# of retries> means indefinite number of retries until success.
-  `attrs` defaults to `"*,+"` to replicate all user and operational attributes



## replicate custom schema

### Method 1 (replicate from master)

`/root/openldap/schema/replicate_custom_schema.ldif`

```
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcSyncrepl
olcSyncRepl: rid=001 
	   provider=ldap://10.113.74.11:389
       bindmethod=simple
       searchbase="cn=schema,cn=config" 
       type=refreshAndPersist
       retry="5 5 300 5" 
       timeout=1
       binddn="cn=Syncer,dc=A063021,dc=nasa"
 	   credentials=hahaYouCatchMe
```

`ldapmodify -Y EXTERNAL -H ldapi:/// -f replicate_custom_schema.ldif`

### Method 2 (manually add)

refer Build custom schema and manually add all needed schema



### reference

- Add modules
  - http://www.zytrax.com/books/ldap/ch6/slapd-config.html#use-modules
  - http://www.zytrax.com/books/ldap/ch6/#moduleload

- replication setting
  - https://www.itzgeek.com/how-tos/linux/configure-openldap-master-slave-replication.html
  - http://www.zytrax.com/books/ldap/ch7/#overview
  - https://www.openldap.org/doc/admin24/replication.html
- more details about syncrepl 
  - https://www.openldap.org/doc/admin24/slapdconfig.html#syncrepl

- replicate custom schema
  - https://www.openldap.org/lists/openldap-technical/201308/msg00062.html
  - https://www.openldap.org/doc/admin24/replication.html

不確定monitor