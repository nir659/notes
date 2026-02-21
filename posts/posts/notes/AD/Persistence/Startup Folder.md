In current user folder, will trigger when current user signs in:

```plaintext
c:\Users\[USERNAME]\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
```

Or in the global startup folder, requires administrative privileges but will trigger as SYSTEM on boot _and_ in a user context whenever any user signs in:

```plaintext
C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp
```