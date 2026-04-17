---
title: Virtual mailbox map drift and stale postmap state
slug: virtual-mailbox-map-drift-and-stale-postmap-state
type: incident
status: draft
date: 2026-02-15
updated: 2026-02-15
tags:
  - mail
  - postfix
  - dovecot
  - virtual-users
summary: New virtual recipients could authenticate in Dovecot but Postfix rejected RCPT because the authoritative mailbox map and compiled hash state were out of sync.
verification_status: verified
verified_on: 2026-02-15
---

# Virtual mailbox map drift and stale postmap state

## Context

I added new virtual recipients such as `dmarc@nir.rip` that could authenticate to IMAP in Dovecot, but Postfix rejected SMTP RCPT with `Recipient address rejected: User unknown in virtual mailbox table`.

This was not an account-creation failure on the IMAP side. It was a drift problem between the mailbox map Postfix was actually using and the file I was editing.

## Goal

- make newly added virtual recipients usable for inbound mail
- ensure Postfix and Dovecot agree on authoritative mailbox state
- verify the active mailbox map returns the expected Maildir paths

## Environment

- Mail stack: Postfix + Dovecot
- Virtual users / virtual mailbox maps
- Postfix hash-backed mailbox map files
- Example affected recipient: `dmarc@nir.rip`

## The Actual Problem

The failure had two parts:

- Postfix was referencing a different virtual mailbox map than the source file I was editing
- the compiled hash database was stale, so even correct edits were not being used until `postmap` rebuilt the `.db`

That meant Dovecot could successfully authenticate the user while Postfix still rejected RCPT because its active mailbox lookup path was wrong.

## Impact

- inbound mail to the new recipients was rejected by Postfix at RCPT
- outbound mail from existing users was unaffected
- existing mailbox recipients already present in the active Postfix map were unaffected
- new recipients added during this change window were not functional for delivery

## Timeline

- 06:46 I observed a successful IMAP login for `dmarc@nir.rip` in Dovecot
- 06:47 I attempted SMTP submission and Postfix rejected RCPT with `User unknown in virtual mailbox table`
- 06:47 I ran a map query and noticed a warning that the map `.db` was older than the source for the file I tested
- 06:48 I rebuilt the map database with `postmap` and reloaded / restarted Postfix
- 06:49 I verified map lookups returned correct Maildir paths for `dmarc@nir.rip` and `xx@nir.rip`
- 06:50 I found the remaining mismatch: I was editing `vmailbox`, but Postfix was actually configured to use `virtual_mailbox_maps`; I fixed this by updating the authoritative file

## The Final State

- new recipients exist in the mailbox map Postfix actually uses
- the compiled `.db` matches the source file
- Postfix accepts RCPT for the new recipients
- Dovecot authentication and Postfix recipient acceptance are aligned again

## What Changed

- updated the authoritative mailbox map at `/etc/postfix/virtual_mailbox_maps`
- rebuilt the hash database with `postmap`
- restarted Postfix so the active process tree reflected the corrected map state

```bash
postmap /etc/postfix/virtual_mailbox_maps
systemctl restart postfix
```

## Verification

Verified lookups:

```bash
postmap -q dmarc@nir.rip /etc/postfix/virtual_mailbox_maps
postmap -q xx@nir.rip /etc/postfix/virtual_mailbox_maps
```

Expected proof:

- both lookups return the expected `nir.rip/<user>/Maildir/` path
- SMTP submission no longer fails at RCPT for the new recipients
- Dovecot auth success and Postfix mailbox acceptance now agree on the same recipients

## Failure Modes Worth Caring About

- editing `vmailbox` while Postfix is actually configured for `virtual_mailbox_maps`
- forgetting to run `postmap` after changing a hash-backed source file
- treating successful IMAP login as proof that SMTP delivery path is correct
- ignoring warnings about stale `.db` files and assuming Postfix is using current source data
