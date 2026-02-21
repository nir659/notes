### **Netexec**

1. **Enumerate Domain Machines for SMB Signing**
```
    nxc smb 192.168.1.0/24
```
- **Validate Credentials**
```
nxc smb 192.168.1.1 -u 'jdoe' -p 'Password123'
```
- **Find Valid Machines for Connection**
```
    nxc smb 192.168.1.0/24 -u 'jdoe' -p 'Password123'
```
- **Enumerate Shared Resources**
```
    nxc smb 192.168.1.1 -u 'jdoe' -p 'Password123' --shares
```
- **Enumerate Users and Groups**
```
    nxc smb 192.168.1.1 -u 'jdoe' -p 'Password123' --users
    nxc smb 192.168.1.1 -u 'jdoe' -p 'Password123' --groups
```
- **Dump LSA and NTDS** If you have domain admin privileges:
```
    nxc smb 192.168.1.1 -u 'jdoe' -p 'Password123' --lsa
    nxc smb 192.168.1.1 -u 'jdoe' -p 'Password123' --ntds
```

### SMBMap

We can enumerate SMB shares and access to system using these command:

```
smbmap -H corp-dc #List share with anonymous access
smbmap -H corp-dc -u "user" -p "P@ssword123!" #List Devan's shares
smbmap -H corp-dc -u "user" --prompt ##List Devan's shares without writing password in cleartext
```

![[image 2.png]]

If the signing of message is disabled we can use it for Relay attacks and potentially of exploit eternalblue vuln.

## SMB Client

Similar to SMBMap, we can use it to enumerate shares and interact with file system prompt

```
smbclient -L //corp-dc -N     #Anonymous Login (-N no credentials)
smbclient //corp-dc -U "dev-angelist.lab/devan%P@ssword123!"     #List Devan's shares
smbclient //corp-dc/SharedFiles -U devan
smbclient //corp-dc/SharedFiles -U "dev-angelist.lab/devan%P@ssword123!" #we can get file shared using get command
#File system prompt includes command such as: cd, dir, ls, get, put
```

![[image 3.png]]

## [PowerHuntShares](https://github.com/NetSPI/PowerHuntShares)

Useful for enumerate shares, discovering sensitive files, ACLs for shares, networks, computers, etc, and generates a nice HTML report.

```
Invoke-HuntSMBShares -NoPing -OutputDirectory C:\AD\Tools -HostList C:\AD\Tools\servers.txt
```

## [SMB Common Attacks](https://dev-angelist.gitbook.io/writeups-and-walkthroughs/homemade-labs/active-directory/smb-common-attacks)

- SMB Tools & Guest or Anonymous access to Shares
- RCE Via access to Administrative Shares
- SMB Brute Forcing
- SMB Password Spraying
- SMBv1 EternalBlue (CVE-2017-0144)
- Net-NTLM Capture Attack
- Pass the Hash Attack (PTH)
- Net-NTLM Relay Attack