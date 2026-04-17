---
title: Mail observability rollout
slug: mail-observability-rollout
type: experiment
status: draft
date: 2026-03-04
updated: 2026-03-04
tags:
  - mail
  - observability
  - grafana
  - alloy
  - loki
  - rollout
summary: Replacing Promtail with Grafana Alloy exposed design problems in mail log ingestion and forced a cleaner journald-first observability model.
verification_status: partial
verified_on: 2026-03-04
---

# Mail observability rollout

## Context

This is the rollout log for the mail logging and archive cleanup work.

Promtail got replaced with Grafana Alloy on the edge nodes and the mail host.

The migration itself was not the interesting part. The interesting part was finding where the design was lying.

## Goal

- replace Promtail with Grafana Alloy on the mail host and edge nodes
- keep mail logging reliable during the migration
- separate live ingestion, archive retention, and archive health into something operationally honest

## Environment

- mail host
- other edge nodes
- Grafana Alloy for log shipping
- Loki for hot query storage
- Grafana for query and dashboard view
- journald as the intended mail source of truth
- node_exporter textfile collector + Prometheus for archive health metrics

## The Actual Problem

The migration stopped being a package swap and turned into design work because several assumptions broke at once.

### 1. Permissions

Alloy initially hit predictable permission failures:

- Docker socket access
- MongoDB log access on another node
- legacy Promtail positions file access

None of that was intellectually interesting. It was just Linux reminding me that “reads logs” still means “needs permission to read logs.”

### 2. Stale backlog replay

The bigger issue was old log replay.

Alloy started ingesting old container history and broad file matches, which caused Loki to reject entries as too far behind.

### 3. Bad path selection

On the mail host, broad mail globs were matching things they had no business matching:

- live mail logs
- rotated logs
- compressed history

That was enough to prove the live source should not be file-based for mail.

## What Changed

### Keep

- journald as the mail source of truth
- Loki as hot query storage
- nginx file ingestion
- plain text archive exports for mail
- cursor-based archive state
- node_exporter textfile metrics for archive health

### Drop or defer

- Promtail
- live ingestion from broad mail file globs
- Docker log ingestion on first cutover
- treating archive freshness as the same thing as semantic correctness

## The Final State

### Live path

- journald -> Alloy -> Loki -> Grafana

### Archive path

- journald -> cursor exporter -> daily postfix / dovecot archive logs

### Archive health path

- archive files + cursor files -> metrics script -> node_exporter textfile collector -> Prometheus -> alert rules

## Verification

What now exists:

- working Alloy on the mail host
- working Alloy on another edge node
- working archive timer
- working archive files
- working archive cursor state
- working textfile metrics for archive freshness
- working Prometheus alert rules for stale archive state

## Failure Modes Worth Caring About

- permissions that silently block log reads on cutover
- stale backlog replay that causes Loki rejection and false “migration failure” symptoms
- broad mail file globs pulling in rotated or compressed history
- no durable daily summaries for mail outcomes
- no clean long-term delivery history or auth summaries tied back to config changes

## Honest Conclusion

The migration is done.

The observability system is not done.

Right now the foundation is real. The next step is to stop collecting only raw evidence and start producing summaries that survive longer than the patience of the person who has to grep them.

## References

- [Grafana Alloy Migration for the Mail Host](../../lab/research/grafana-alloy-mail-migration.md)
- [Mail Observability Pipeline](../../lab/primitives/mail-observability-pipeline.md)
- [Mail Log Archival](../../lab/primitives/Mail%20Log%20Archival.md)
- [From Pipes to Signal: Building a Mail Observability System](./2026-03-05-mail-observability-from-pipes-to-signal.md)
