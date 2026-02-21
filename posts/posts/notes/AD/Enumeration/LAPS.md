The Local Administrative Password Solution (LAPS) is Microsoft’s product for managing local admin passwords in the context of an Active Directory domain. It frequently generates strong and unique passwords for the local admin users of enrolled machines. This password property and its expiry time are then written to the computer object in Active Directory. Read access to LAPS passwords is only granted to Domain Admins by default, but often delegated to special groups.

The permission `ReadLAPSPassword` grants users or groups the ability to read the `ms-Mcs-AdmPwd` property and as such get the local admin password. You can look for this property using e.g. BloodHound or PowerView. We can also use PowerView to read the password, if we know that we have the right `ReadLAPSPassword` privilege to a machine.

```powershell
Get-DomainComputer -identity LAPS-COMPUTER -properties ms-Mcs-AdmPwd
```

We can also use [LAPSToolkit.ps1](https://github.com/leoloobeek/LAPSToolkit/blob/master/LAPSToolkit.ps1) to identify which machines in the domain use LAPS, and which principals are allowed to read LAPS passwords. If we are in this group, we can get the current LAPS passwords using this tool as well.

```powershell
# Get computers running LAPS, along with their passwords if we're allowed to read those
Get-LAPSComputers

# Get groups allowed to read LAPS passwords
Find-LAPSDelegatedGroups
```