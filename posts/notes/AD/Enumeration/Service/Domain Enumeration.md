## **PowerView Usage**

- **Start InviShell** (using cmd)
```
    C:\InviShell\RunWithRegistryNonAdmin.bat
```
- **Start PowerView** (using powershell, if you've run InviShell powershell It's already running)
```
    . C:\Powerview.ps1
```
- **Get Domain Information**
```
    Get-NetDomain
```
    Retrieves information about the current domain.
- **Enumerate Domain Controllers**
```
    Get-NetDomainController
```
    Lists all Domain Controllers in the current domain.
- **List Domain Users**
```
    Get-NetUser
```
    Displays all users in the domain, along with detailed attributes.
- **Find High-Value Targets**
```
    Get-NetUser -AdminCount 1
```
    Lists all users flagged as administrators.
- **Enumerate Domain Groups**
```
Get-NetGroup
```
Retrieves all domain groups.
```
    Get-NetGroupMember -GroupName "Domain Admins"
```
Lists members of the "Domain Admins" group.
- **Locate Domain Computers**
```
    Get-NetComputer
```
Lists all computers in the domain.
- **Analyze Trust Relationships**
```
    Get-NetDomainTrust
```
Displays trust relationships between domains.
- **Check ACLs on AD Objects**
```
    Get-ObjectAcl -SamAccountName "Administrator" -ResolveGUIDs
```
Shows ACLs for a specific user account, resolving GUIDs to human-readable names.
- **Find Shares on Domain Computers**
```
    Invoke-ShareFinder
```
Locates shared folders across domain computers.
- **Identify Delegation Configurations**
```
Get-NetUser -SPN
```
Finds user accounts with Service Principal Names (SPNs), often used in Kerberos-based attacks.