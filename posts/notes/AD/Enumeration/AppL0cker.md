Identify the local AppLocker policy. Look for exempted binaries or paths to bypass.

```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

PowerShell

Get a remote AppLocker policy, based on the Distinguished Name of the respective Group Policy (you could identify this e.g. in BloodHound).

```powershell
Get-AppLockerPolicy -Domain -LDAP "LDAP://targetdomain.com/CN={16641EA1-8DD3-4B33-A17F-9F259805B8FF},CN=Policies,CN=System,DC=targetdomain,DC=com"  | select -expandproperty RuleCollections
```

PowerShell

Some high-level bypass techniques:

- Use [LOLBAS](https://lolbas-project.github.io/) if only (Microsoft-)signed binaries are allowed.
- If binaries from `C:\Windows` are allowed (default behavior), try dropping your binaries to `C:\Windows\Temp` or `C:\Windows\Tasks`. If there are no writable subdirectories but writable files exist in this directory tree, write your file to an alternate data stream (e.g. a JScript script) and execute it from there.
- Wrap your binaries in a DLL file and execute them with `rundll32` to bypass executable rules if DLL execution is not enforced (default behavior).
- If binaries like Python are allowed, use those. If that doesn’t work, try other techniques such as wrapping JScript in a HTA file or running XSL files with `wmic`.
- Otherwise elevate your privileges. AppLocker rules are most often not enforced for (local) administrative users.