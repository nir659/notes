> **System Name:** Mail Stack Boundary Design
> **Domain / Service:** nir.rip
> **Status:** Stable
> **Last Updated:** 2026-02-03

---
## 1. What This Is

This document defines the boundaries of my mail stack now that IMAP and mailbox storage live on the VPS (single-host setup). This is not an installation guide. It is a boundary and risk document: what is exposed, what is trusted, what is isolated, and what I am knowingly accepting.

---
## 2. Design Intent

- Minimize public attack surface without harming deliverability.
- Make trust relationships explicit and reviewable.
- Keep failure modes understandable under stress.
- Preserve a clean escape hatch back to a split-host design later.

---
## 3. System Boundaries

### 3.1 External Boundary

Everything outside the VPS is untrusted.

Public entrypoints:
- SMTP (25) — server-to-server
- Submission (587) — authenticated client send
- IMAPS (993) — authenticated client read

### 3.2 Internal Boundary

With everything on one VPS, boundaries are enforced by:
- Unix permissions
- service users/groups
- socket access
- firewall rules
- configuration correctness

This is weaker than physical or VM separation. That is a known compromise.

---
## 4. Trust Model

### Trust tiers
1. **Untrusted:** Internet (random MTAs, scanners, bots)
2. **Semi-trusted:** Legitimate external MTAs
3. **Trusted:** Authenticated users
4. **Highly trusted:** Root and local service accounts

### Assumptions
- External senders can be compromised.
- Authenticated users can lose credentials.
- Client devices are not trustworthy.
- VPS compromise equals total system compromise.

---
## 5. Public Attack Surface

### 5.1 SMTP (25)

**Purpose:** Receive inbound mail.

Primary risks:
- Open relay probing
- SMTP abuse and flooding
- Queue growth and disk exhaustion
- Reputation poisoning
- MTA parsing vulnerabilities

Controls:
- Strict relay and recipient restrictions
- Recipient existence checks via maps
- Rate limiting
- Opportunistic TLS
- Queue visibility

### 5.2 Submission (587)

**Purpose:** Authenticated outbound mail.

Primary risks:
- Credential stuffing
- Stolen credentials → spam → reputation death
- Weak TLS or auth downgrade

Controls:
- Mandatory auth + TLS
- Fail2ban on auth failures
- Separate submission listener
- Strong credentials
- Per-user rate limiting

### 5.3 IMAPS (993)

**Purpose:** Mailbox access.

Primary risks:
- Brute force auth
- IMAP implementation flaws
- Client compromise
- Credential reuse

Controls:
- TLS-only access
- Fail2ban on dovecot auth
- Disable legacy auth methods
- Patch discipline

---
## 6. Internal Interfaces & Blast Radius

### 6.1 Postfix ↔ Dovecot (LMTP)

Risks:
- Misdelivery via config error
- Permission mistakes exposing mailboxes
- Over-broad socket access

Controls:
- Unix socket only
- Minimal permissions
- Dedicated virtual mail user/group

### 6.2 Mail Storage (Maildir)

Risks:
- Disk exhaustion kills all services
- Backup failure causes silent data loss
- Ownership drift leaks data

Controls:
- Disk usage monitoring
- Quotas
- Tested backups

---
## 7. Boundary Weakness: Single-Host Reality

By collapsing IMAP, storage, and MTA onto one VPS:

- A single RCE compromises mail, credentials, and identity.
- Disk pressure impacts all services simultaneously.
- Isolation relies on discipline, not hardware.

---
## 8. Observability Philosophy

I expect to clearly see **availability failures** and **auth failures**: service down, queues backing up, disk filling, brute-force attempts, and obvious delivery failures (bounces, Gmail rejections). I knowingly accept being blind to **quiet failures**: slow reputation decay, subtle spam-folder placement, low-volume credential abuse, and partial message loss that does not trigger errors. Metrics are intentionally minimal because email systems generate noise that encourages false confidence. Human checks (sending test mail, checking logs, verifying queue state) surface meaningful failures faster than dashboards in a system of this size.

---
## 9. Falsifiability: What Would Prove This Design Insufficient

This design is considered **invalid** if any of the following become true:

- A single compromised user credential can generate enough spam to irreversibly damage domain or IP reputation.
- Disk exhaustion occurs without early human-visible warning.
- Abuse or compromise is detected only after third-party complaints.
- Mailbox data loss occurs without immediate detection.
- Operational load exceeds what can be safely handled without automation.

If any of these happen, the boundary design has failed and must be replaced with stronger isolation (split-host or tiered design).

---
## 10. Failure & Abuse Scenarios

### Most likely
- Brute force against 993/587
- Spam flood against 25
- DNS drift breaking deliverability
### Worst case
- VPS compromise → mailbox exfiltration → outbound abuse → reputation burn
### 3am recovery plan
1. Restrict inbound ports immediately.
2. Check disk and queue state.
3. Identify abuse in logs.
4. Rotate credentials.
5. Pause outbound if reputation is damaged.

---
## 11. Improvements / Next Iteration

- Re-separate edge MTA and mailbox storage.
- Add lightweight alerts for disk and queue depth.
- Enforce stricter per-user limits.
- Implement MTA-STS and TLS reporting.

---
## 12. Notes I Don’t Want To Relearn

- Email fails quietly.
- Reputation damage is harder to fix than outages.
- Single-host mail stacks work until they don’t.

---
## 13. References

- Postfix documentation
- Dovecot documentation
- Relevant SMTP/IMAP RFCs