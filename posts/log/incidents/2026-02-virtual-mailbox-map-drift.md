> **Date:** 2026-02-15  
> **System:** Mail Stack (Postfix + Dovecot, virtual users)  
> **Severity:** Medium  
> **Status:** Resolved  

---
## 1. Summary

I added new virtual recipients (e.g. `dmarc@nir.rip`) that could authenticate to IMAP in Dovecot, but Postfix rejected SMTP RCPT with “User unknown in virtual mailbox table.”

---

## 2. Detection

- I saw a mail client send attempt fail with: `Recipient address rejected: User unknown in virtual mailbox table`.

- I confirmed in the Postfix submission log that the reject occurred at RCPT stage:
    - `NOQUEUE: reject: RCPT ... 550 5.1.1 <dmarc@nir.rip>: Recipient address rejected: User unknown in virtual mailbox table`

- I checked Dovecot logs and saw successful IMAP logins for `dmarc@nir.rip`, which confirmed the account existed on the IMAP side.

---

## 3. Impact

- Inbound mail to the new recipients was rejected by Postfix at RCPT.
- Outbound mail from existing users was unaffected.
- Existing mailbox recipients already present in the active Postfix map were unaffected.
- New recipients I added during this change window were not functional for delivery.

---

## 4. Root Cause

- Postfix was referencing a different virtual mailbox map than the file I was editing, and the active hash database was stale.

- I also confirmed that when I edit Postfix hash maps, I must rebuild the compiled `.db` with `postmap`; until I do that, Postfix continues using stale data.

---

## 5. Timeline

- 06:46 I observed a successful IMAP login for `dmarc@nir.rip` (Dovecot).
- 06:47 I attempted SMTP submission and Postfix rejected RCPT with “User unknown in virtual mailbox table”.
- 06:47 I ran a map query and noticed a warning that the map `.db` was older than the source for the file I tested.
- 06:48 I rebuilt the map database with `postmap` and reloaded/restarted Postfix.
- 06:49 I verified map lookups returned correct Maildir paths for `dmarc@nir.rip` and `xx@nir.rip`.
- 06:50 I found the remaining mismatch: I was editing `vmailbox`, but Postfix was actually configured to use `virtual_mailbox_maps`; I fixed this by updating the configured map file (and ensuring I edited the authoritative one).

---

## 6. Resolution

- I ensured the new recipients existed in the map file Postfix actually uses (`/etc/postfix/virtual_mailbox_maps`).
- I rebuilt the hash database and restarted Postfix:
```bash
  postmap /etc/postfix/virtual_mailbox_maps
  systemctl restart postfix
```

Verified lookups:
```bash
postmap -q dmarc@nir.rip /etc/postfix/virtual_mailbox_maps
postmap -q xx@nir.rip /etc/postfix/virtual_mailbox_maps
```

(Both returned expected `nir.rip/<user>/Maildir/` paths.)