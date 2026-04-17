---
title: Mail Stack Boundary Design
slug: mail-stack-boundary-design
type: design
status: draft
date: 2026-02-03
updated: 2026-02-03
tags:
  - mail
  - security
  - boundary
  - postfix
  - dovecot
summary: This note defines the trust boundaries, exposed services, and known compromises in the current single-host mail stack.
verification_status: partial
verified_on: 2026-02-03
---

# Mail Stack Boundary Design

## Context

This document defines the boundaries of the mail stack now that IMAP and mailbox storage live on the VPS in a single-host setup.

This is not an installation guide. It is a boundary and risk document: what is exposed, what is trusted, what is isolated, and what is knowingly accepted.

## Goal

- minimize public attack surface without harming deliverability
- make trust relationships explicit and reviewable
- keep failure modes understandable under stress
- preserve a clean escape hatch back to a split-host design later

## Environment

- system name: Mail Stack Boundary Design
- domain / service: `nir.rip`
- status: stable
- single VPS hosting MTA, IMAP, and mailbox storage
- public entrypoints on SMTP 25, Submission 587, and IMAPS 993

## System Boundaries

### External boundary

Everything outside the VPS is untrusted.

Public entrypoints:

- SMTP (25) for server-to-server mail
- Submission (587) for authenticated client send
- IMAPS (993) for authenticated client read

### Internal boundary

With everything on one VPS, boundaries are enforced by:

- Unix permissions
- service users and groups
- socket access
- firewall rules
- configuration correctness

This is weaker than physical or VM separation. That is a known compromise.

## Trust Model

### Trust tiers

1. **Untrusted:** Internet, including random MTAs, scanners, and bots
2. **Semi-trusted:** legitimate external MTAs
3. **Trusted:** authenticated users
4. **Highly trusted:** root and local service accounts

### Assumptions

- external senders can be compromised
- authenticated users can lose credentials
- client devices are not trustworthy
- VPS compromise equals total system compromise

## Public Attack Surface

### SMTP (25)

**Purpose:** Receive inbound mail.

Primary risks:

- open relay probing
- SMTP abuse and flooding
- queue growth and disk exhaustion
- reputation poisoning
- MTA parsing vulnerabilities

Controls:

- strict relay and recipient restrictions
- recipient existence checks via maps
- rate limiting
- opportunistic TLS
- queue visibility

### Submission (587)

**Purpose:** Authenticated outbound mail.

Primary risks:

- credential stuffing
- stolen credentials leading to spam and reputation damage
- weak TLS or auth downgrade

Controls:

- mandatory auth and TLS
- Fail2ban on auth failures
- separate submission listener
- strong credentials
- per-user rate limiting

### IMAPS (993)

**Purpose:** Mailbox access.

Primary risks:

- brute-force auth
- IMAP implementation flaws
- client compromise
- credential reuse

Controls:

- TLS-only access
- Fail2ban on dovecot auth
- disable legacy auth methods
- patch discipline

## Internal Interfaces And Blast Radius

### Postfix to Dovecot (LMTP)

Risks:

- misdelivery via config error
- permission mistakes exposing mailboxes
- over-broad socket access

Controls:

- Unix socket only
- minimal permissions
- dedicated virtual mail user and group

### Mail storage (Maildir)

Risks:

- disk exhaustion kills all services
- backup failure causes silent data loss
- ownership drift leaks data

Controls:

- disk usage monitoring
- quotas
- tested backups

## The Final State

By collapsing IMAP, storage, and MTA onto one VPS:

- a single RCE compromises mail, credentials, and identity
- disk pressure impacts all services simultaneously
- isolation relies on discipline, not hardware

That is the current reality, not a hidden detail.

## Observability Philosophy

I expect to clearly see **availability failures** and **auth failures**: service down, queues backing up, disk filling, brute-force attempts, and obvious delivery failures such as bounces or Gmail rejections.

I knowingly accept being blind to **quiet failures**: slow reputation decay, subtle spam-folder placement, low-volume credential abuse, and partial message loss that does not trigger errors.

Metrics are intentionally minimal because email systems generate noise that encourages false confidence. Human checks like sending test mail, checking logs, and verifying queue state surface meaningful failures faster than dashboards in a system of this size.

## Verification

This note is a boundary and risk document, so its current verification is conceptual:

- the exposed ports and services are explicitly enumerated
- the trust model and control model are written down
- the single-host compromise is documented instead of hidden

## Failure Modes Worth Caring About

### Most likely

- brute force against 993 or 587
- spam flood against 25
- DNS drift breaking deliverability

### Worst case

- VPS compromise leading to mailbox exfiltration, outbound abuse, and reputation burn

### 3am recovery plan

1. Restrict inbound ports immediately.
2. Check disk and queue state.
3. Identify abuse in logs.
4. Rotate credentials.
5. Pause outbound if reputation is damaged.

## What Would Prove This Design Insufficient

This design is invalid if any of the following become true:

- a single compromised user credential can generate enough spam to irreversibly damage domain or IP reputation
- disk exhaustion occurs without early human-visible warning
- abuse or compromise is detected only after third-party complaints
- mailbox data loss occurs without immediate detection
- operational load exceeds what can be safely handled without automation

If any of these happen, the boundary design has failed and should be replaced with stronger isolation.

## Improvements And Next Iteration

- re-separate edge MTA and mailbox storage
- add lightweight alerts for disk and queue depth
- enforce stricter per-user limits
- implement MTA-STS and TLS reporting

## Notes I Do Not Want To Relearn

- email fails quietly
- reputation damage is harder to fix than outages
- single-host mail stacks work until they do not

## References

- Postfix documentation
- Dovecot documentation
- Relevant SMTP and IMAP RFCs
- [Mail Observability Pipeline](./mail-observability-pipeline.md)
- [From Pipes to Signal: Building a Mail Observability System](../../log/raw/2026-03-05-mail-observability-from-pipes-to-signal.md)
