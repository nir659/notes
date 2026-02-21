Sometimes you may find yourself in a PowerShell session that enforces Constrained Language Mode (CLM). This is often the case when you’re operating in an environment that enforces AppLocker (see above).

You can identify you’re in constrained language mode by polling the following variable to get the current language mode. It will say `FullLanguage` for an unrestricted session, and `ConstrainedLanguage` for CLM. There are other language modes which I will not go into here.

```powershell
$ExecutionContext.SessionState.LanguageMode
```

PowerShell

The constraints posed by CLM will block many of your exploitations attempts as key functionality in PowerShell is blocked. Bypassing CLM is largely the same as bypassing AppLocker as discussed above. Another way of bypassing CLM is to bypass AppLocker to execute binaries that execute a custom PowerShell runspace (e.g. [Stracciatella](https://github.com/mgeeky/Stracciatella)) which will be unconstrained.

Another quick and dirty bypass is to use in-line functions, which sometimes works. If e.g. `whoami` is blocked, try the following:

```powershell
&{whoami}
```