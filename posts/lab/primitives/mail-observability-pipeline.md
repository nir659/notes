---
title: Mail Observability Pipeline
slug: mail-observability-pipeline
type: design
status: draft
date: undated
updated: 2026-04-17
tags:
  - mail
  - observability
  - grafana
  - loki
  - alloy
summary: The mail logging system is structured as a hot journald-to-Loki path, a warm archive layer, and a future cold summary layer.
verification_status: partial
verified_on:
---

# Mail Observability Pipeline

## Context

This note documents the shape of the mail logging system as it exists now.

The main decision is simple: journald is the live source of truth for mail, Loki is the hot query layer, and exported archive files are the warm forensic layer. The point is to stop pretending raw logs and long-term history are the same thing.

Mail systems produce three different kinds of data at once:

- legitimate flow: accepted, queued, deferred, bounced, delivered
- background noise: auth attempts, random probes, cheap scanners, UFW blocks
- internal health: queue behaviour, warnings, service state, config drift

If all of that gets treated as one endless blob of text, it becomes expensive noise. The system needs separation by role.

## Goal

- keep journald as the authoritative live source for mail
- provide hot search in Loki and Grafana
- retain grep-friendly archive files outside Loki
- build toward structured summaries that outlive raw logs

## Environment

- mail host with Postfix, Dovecot, and Nginx
- journald as authoritative live source
- Grafana Alloy with `loki.source.journal` and `loki.source.file`
- Loki and Grafana for hot-path query and dashboards
- archive service writing daily logs and cursor files
- Prometheus and node_exporter textfile metrics for archive health

## The Final State

```mermaid
flowchart TD
    A[Mail Host - Postfix + Dovecot + Nginx] --> B[journald - authoritative live source]
    A --> C[/var/log/nginx/*.log - nginx file logs]

    B --> D[Grafana Alloy - loki.source.journal]
    C --> E[Grafana Alloy - loki.source.file nginx]
    D --> F[Loki]
    E --> F
    F --> G[Grafana]

    B --> H[mail-journal-archive.service]
    H --> I[/var/log/journal-archive/postfix/postfix-YYYY-MM-DD.log]
    H --> J[/var/log/journal-archive/dovecot/dovecot-YYYY-MM-DD.log]
    H --> K[/var/lib/journal-cursors/postfix.cursor]
    H --> L[/var/lib/journal-cursors/dovecot.cursor]

    I --> M[mail-archive-metrics.service]
    J --> M
    K --> M
    L --> M
    M --> N[/var/lib/node_exporter/textfile_collector/mail_archive.prom]
    N --> O[node_exporter]
    O --> P[Prometheus]
    P --> Q[Alert rules]
    P --> G
```

### Hot layer

- journald
- Grafana Alloy
- Loki
- Grafana

Purpose:
- live debugging
- recent searches
- operational alerts

### Warm layer

- plain text exported mail archives under `/var/log/journal-archive/`
- cursor state under `/var/lib/journal-cursors/`

Purpose:
- grep-friendly incident review
- low-cost retention
- independent copy outside Loki

### Cold layer

Not fully built yet.

This is where daily structured summaries should live:

- deliveries by destination domain
- deferred and bounced counts
- dovecot auth failures by source IP / username
- queue and warning trend summaries

That layer is the one that will still matter years later.

## Design Rules

1. journald is the live source of truth for mail
2. archive files are forensics, not live ingestion
3. Loki is for hot search, not eternal storage
4. summary data should outlive raw data
5. config and topology changes should be tracked alongside logs

## Non-Goals

This design is not trying to:

- store every raw line forever in Loki
- treat rotated mail files as authoritative
- turn Docker backlog into fake observability
- pretend archive freshness is the same as semantic correctness

## Verification

Current evidence for the design:

- archive files and cursor files exist as distinct outputs
- Alloy has separate journal and file paths by source type
- Prometheus and textfile metrics are part of the designed health path

This note is a design note, so verification is architectural rather than command-driven.

## Failure Modes Worth Caring About

- archive timer stops firing
- cursor stops advancing
- archive files stop growing
- Alloy stops shipping journal events
- label or path changes silently duplicate ingestion
- summaries get skipped, leaving only raw text behind

## What Would Prove This Design Insufficient

Any of the following would be enough:

- I cannot answer which recipient domains started deferring more over time
- I cannot tell whether auth failures are random background noise or focused abuse
- I cannot correlate config changes with delivery regressions
- archive files continue to exist but are incomplete or duplicated
- Loki is the only place recent history exists

If that happens, the design is not observability. It is just storage with better marketing.

## References

- [Mail Stack Boundary Design](./Mail-Stack%20Boundary%20Design.md)
- [Mail Log Archival](./Mail%20Log%20Archival.md)
- [Grafana Alloy Migration for the Mail Host](../research/grafana-alloy-mail-migration.md)
- [Mail Observability Rollout](../../log/raw/2026-03-04-mail-observability-rollout.md)
- [From Pipes to Signal: Building a Mail Observability System](../../log/raw/2026-03-05-mail-observability-from-pipes-to-signal.md)
