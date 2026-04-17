---
title: Grafana Alloy Migration for the Mail Host
slug: grafana-alloy-mail-migration-for-the-mail-host
type: research
status: draft
date: undated
updated: 2026-04-17
tags:
  - mail
  - observability
  - grafana
  - alloy
  - promtail
summary: Migrating from Promtail to Grafana Alloy forced the mail host to move to a journal-first design and drop stale file-glob ingestion paths.
verification_status: partial
verified_on:
---

# Grafana Alloy Migration for the Mail Host

## Context

This note documents the migration away from Promtail on the mail host and the reasons behind the final layout.

Promtail worked, but it was the wrong thing to keep investing in. The migration itself also exposed a more important issue: old file globs and container backlog are capable of poisoning a log pipeline faster than the agent choice can save it.

The initial bad assumptions were:

- file globs like `/var/log/mail.*` are safe
- first-run backfill is harmless
- rotated and compressed files can sit in the same live path without consequences
- Docker is worth keeping on the first cutover

Those assumptions were wrong.

## Goal

- replace Promtail with Grafana Alloy on the mail host
- remove stale or misleading live log paths
- align live ingestion with the real source of truth for mail services

## Environment

- mail host
- Promtail as the old agent
- Grafana Alloy as the replacement
- Postfix and Dovecot as mail services
- Nginx as a file-tailed service still worth keeping in the live path
- Loki as the downstream storage target

## The Actual Problem

The migration was not just about changing the agent. It exposed design problems in the ingestion path:

- mail ingestion depended on file paths broad enough to match rotated and compressed history
- old backlog could trigger Loki rejection with `too far behind`
- convenience paths were being treated as authoritative sources

## What Changed

### Before

- Promtail shipped journal, docker, nginx, and postfix-style file logs
- mail ingestion depended on file paths broad enough to match rotated and compressed history
- old backlog could trigger Loki rejection with `too far behind`

### After

- Grafana Alloy replaced Promtail
- `loki.source.journal` became the live mail source for Postfix and Dovecot
- `loki.source.file` was kept only for Nginx
- Docker was deferred on first cutover
- broad mail file globs were removed from the live path

## The Final State

journald was already the real source of truth for Postfix and Dovecot. Tailing `mail.log` in parallel was just another way to manufacture duplicates, replay stale files, and confuse the system about what “live” means.

The final rule is simple:

- mail: journal
- nginx: file
- archive: journal export only

### Live into Loki

- `loki.source.journal` for Postfix
- `loki.source.journal` for Dovecot
- `loki.source.journal` for broader system context
- `loki.source.file` for Nginx only

### Not in live path

- rotated mail files
- compressed mail history
- archive exports
- Docker backlog

## Why This Version Is Better

- fewer duplicate paths
- less stale replay risk
- labels now describe real sources instead of path accidents
- easier reasoning during incidents
- closer alignment between where the logs come from and what the system actually does

## Verification

Current evidence from the migration:

- Grafana Alloy replaced Promtail on the mail host
- `loki.source.journal` is the active live source for Postfix and Dovecot
- file-tail ingestion is retained only for Nginx
- broad mail file globs and Docker backlog were removed from the first cutover

## Failure Modes Worth Caring About

- trusting converted config output as finished design
- letting first-run backfill replay stale streams into Loki
- treating wildcards as harmless when they span rotated or compressed history
- confusing convenience file paths with authoritative service log sources

## Next Steps

Migration is plumbing. It is not the final product.

The next real work is:

- monitor archive freshness
- produce daily mail summaries
- track config and topology changes
- decide what deserves long retention as structured data

That is where the system becomes useful instead of merely operational.

## References

- [Mail Observability Pipeline](../primitives/mail-observability-pipeline.md)
- [Mail Log Archival](../primitives/Mail%20Log%20Archival.md)
- [Mail Observability Rollout](../../log/raw/2026-03-04-mail-observability-rollout.md)
- [From Pipes to Signal: Building a Mail Observability System](../../log/raw/2026-03-05-mail-observability-from-pipes-to-signal.md)
