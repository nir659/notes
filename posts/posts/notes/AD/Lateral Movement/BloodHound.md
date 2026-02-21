Use `Invoke-BloodHound` from `SharpHound.ps1`, or use `SharpHound.exe`. Both can be run reflectively, get them [here](https://github.com/BloodHoundAD/BloodHound/tree/master/Collectors). Examples below use the PowerShell variant but arguments are identical.

```powershell
# Run all checks, including restricted groups enforced through the domain  🚩
Invoke-BloodHound -CollectionMethod All,GPOLocalGroup

# Running LoggedOn separately sometimes gives you more sessions, but enumerates by looping through hosts so is VERY noisy 🚩
Invoke-BloodHound -CollectionMethod LoggedOn
```

PowerShell

For real engagements definitely look into the [various arguments](https://bloodhound.readthedocs.io/en/latest/data-collection/sharphound-all-flags.html) that BloodHound provides for more stealthy collection and exfiltration of data.