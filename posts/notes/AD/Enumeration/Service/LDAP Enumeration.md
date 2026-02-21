### `LdapSearch`

Perform anonymous or credentialed enumeration of the LDAP directory:

```
ldapsearch -H ldap://192.168.1.1 -x -s base namingcontexts
ldapsearch -H ldap://192.168.1.1 -D 'jdoe@DC.LOCAL' -w 'Password123' -x -b "DC=DC,DC=LOCAL"
```

### `LdapDomainDump`

Dump LDAP data in JSON and HTML formats for easier analysis:

```
ldapdomaindump -u 'DC.LOCAL\jdoe' -p 'Password123' 192.168.1.1
```

### DNS Enumeration

Resolve DNS name using nslookup for retrieving useful info regarding target:

```
nslookup -type=SRV DC.LOCAL
```