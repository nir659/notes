---
title: From Pipes to Signal Building a Mail Observability System
slug: from-pipes-to-signal-building-a-mail-observability-system
type: design
status: draft
date: 2026-03-05
updated: 2026-03-05
tags:
  - mail
  - observability
  - grafana
  - loki
  - alloy
  - postfix
  - dovecot
  - homelab
summary: Mail observability became a multi-layer design built around journald, Alloy, archive exports, summaries, and health metrics instead of raw log collection alone.
verification_status: partial
verified_on: 2026-03-05
---
# From Pipes to Signal: Building a Mail Observability System

## Context

I stopped treating logging like “dump everything somewhere and hope future me sorts it out.”

This started as a normal setup problem. I had a mail server, logs, Loki on another host, and Promtail still hanging around even though Grafana had already made it clear Promtail was being phased out. At first, the goal was simple: get logs off the box, make them searchable, and avoid breaking anything.

That part was easy.

The harder part was realising that “logs exist” is not the same as “I have observability.”

## Goal

For the mail stack, I wanted four things:

1. **Live searchability** so I could inspect Postfix, Dovecot, and Nginx in Grafana.
2. **Durable archives** so I still had readable history if Loki retention was short.
3. **Derived summaries** so I could answer useful questions without grepping forever.
4. **Health checks for the logging pipeline** so I would know if archive or summary jobs silently died.

That turned into a multi-layer pipeline, not just “ship journal to Loki.”

## Environment

- mail host running Postfix, Dovecot, and Nginx
- journald as the live event source for mail services
- Grafana Alloy for collection and shipping
- Loki on a separate logs host
- Grafana for search and dashboards
- archive jobs exporting journal entries into daily text files
- summary jobs deriving smaller CSV datasets from the archives
- node_exporter textfile metrics scraped by Prometheus

## The Final State

The mail host now looks like this:

- **Postfix + Dovecot** log to **journald**
- **Grafana Alloy** reads mail logs with `loki.source.journal`
- **Nginx** is still tailed from files with `loki.source.file`
- logs go over WireGuard to **Loki** on the logs host
- a separate archive job exports Postfix and Dovecot journal entries into daily text files
- summary jobs derive smaller CSV datasets from those archives
- a metrics exporter publishes freshness and file size metrics into `node_exporter`’s textfile collector
- **Prometheus** scrapes those metrics and alert rules check for stale pipelines
- **Grafana** shows everything in one dashboard

So the system is no longer just “logs go to Loki.”

It is now:

- **hot path** -> journald -> Alloy -> Loki
- **warm path** -> journald -> archive files
- **signal path** -> archive files -> daily summaries
- **meta-observability path** -> metrics exporter -> node_exporter -> Prometheus -> Grafana / alerts

That is a more honest design.

## What Changed

### Why I moved off file-based mail tails

At one point I was tailing `mail.log` and hit exactly the mess I should have expected:

- old rotated logs being picked up
- compressed historical files getting matched
- Loki rejecting old entries as too far behind
- duplicate-looking streams from overlapping sources

The fix was to stop pretending file tails were the source of truth for mail on this box.

They were not.

`journald` was the real source of truth for Postfix and Dovecot, so the clean move was:

- keep **Postfix / Dovecot** on `loki.source.journal`
- keep **Nginx** on `loki.source.file`
- keep **archive files out of live ingestion**

That removed the replay mess and stopped duplicate ingestion.

### The archive layer

I did not want journald to be the only place history lived.

So I built a small cursor-based archive job that exports:

- `postfix-YYYY-MM-DD.log`
- `dovecot-YYYY-MM-DD.log`

from journald into plain text files.

I used journald cursors instead of time slicing because I wanted:

- safe reruns
- minimal duplicates
- clean recovery after reboots
- explicit state

The archive job runs every 5 minutes, and cursor files are stored separately from the log files.

That gave me a warm storage layer that is:

- human-readable
- compressible
- grep-friendly
- independent of Loki retention

### The summaries

This is where the system stopped being “just more logs.”

I built three daily summaries.

#### 1. Postfix delivery summary

This answers:

- which destination domains got mail
- how many were delivered
- how many deferred
- how many bounced

Example:

```csv
date,domain,delivered,deferred,bounced
2026-03-04,gmail.com,1,0,0
2026-03-04,nir.rip,3,0,0
```

This was the first actual **flow** summary.

#### 2. Dovecot auth summary

This answers:

- which users authenticated
- from which source IP
- how many successes and failures

Example:

```csv
date,username,src_ip,success_count,failure_count
2026-03-04,contact@remitae.com,100.69.185.58,6,0
2026-03-04,dmarc@nir.rip,100.69.185.58,6,0
2026-03-04,ly@nir.rip,100.69.185.58,8,0
2026-03-04,nir@nir.rip,100.69.185.58,6,0
```

This was useful for **baseline internal access visibility**, but not very useful for attack or noise visibility.

That was an important lesson: not every log source is equally useful for every question.

#### 3. Postfix auth-noise summary

This became the useful security and noise summary.

It aggregates SMTP auth and probe patterns by source IP from Postfix archive logs:

- disconnects with `auth=0/1`
- lost connections after AUTH
- EHLO + quit style probes

Example:

```csv
date,src_ip,auth_disconnect_count,lost_after_auth_count,quit_after_ehlo_count,total_probe_events
2026-03-04,103.81.170.109,29,0,29,29
2026-03-04,158.94.211.170,30,30,0,60
2026-03-04,193.32.162.196,1,0,1,1
```

This immediately exposed repeat noisy IPs and daily hostile patterns in a way raw logs did not.

## Mistakes I Made

### 1. I duplicated my own logs

I originally had:

- a broad system journal source
- a dedicated Postfix journal source
- a dedicated Dovecot journal source

Then I filtered by `host` in Grafana and got duplicate lines. Grafana was not broken. I was ingesting the same events twice.

That got fixed by removing the broad generic journal source for the mail host.

### 2. I got confused by Grafana’s alerting UI

The Prometheus alert rules were fine. Grafana’s “Health unknown / No data” display for data source-managed rules made it look more broken than it was.

Prometheus was the real source of truth:

- rules loaded
- health `ok`
- state `inactive`

So that was a UI misunderstanding, not an alerting failure.

### 3. I wrote a parser that extracted PIDs instead of IPs

In the first version of the Postfix auth-noise summary, I lazily matched the first bracketed value on a line.

That captured:

- `postfix/smtpd[815175]`

instead of:

- `unknown[158.94.211.170]`

So I got rows full of PIDs pretending to be source IPs. Completely useless.

That got fixed by only extracting bracketed values from connection phrases like:

- `connect from ...[IP]`
- `disconnect from ...[IP]`
- `lost connection after AUTH from ...[IP]`

That was just sloppy parsing.

## Verification

The part I like most is that I did not stop at generating logs and summaries.

I also built **health metrics for the pipeline itself** using `node_exporter`’s textfile collector.

That exporter publishes things like:

- archive file mtime
- cursor file mtime
- summary file mtime
- archive file size
- summary file size

Prometheus scrapes those metrics, and Grafana now shows:

- archive freshness
- cursor freshness
- summary freshness
- file sizes
- exporter up / down

So now I can tell the difference between:

- “mail is quiet”
- “summary job is stale”
- “archive is updating but summary is lagging”
- “the exporter itself is dead”

That is the difference between having data and having a system.

### What the dashboard showed me immediately

As soon as I built the dashboard, the setup became easier to trust.

A few examples:

- archive freshness stayed low and stable, which meant the journal export loop was healthy
- summary jobs showed the expected sawtooth pattern because they run slower than the archive job
- Dovecot summary age was visibly slower than the others, which matched the timer interval
- file sizes gave me a quick signal that jobs were actually producing output

## Failure Modes Worth Caring About

- duplicate ingestion from overlapping journal or file sources
- stale archive or summary jobs that quietly stop producing fresh data
- parsers that emit syntactically valid but semantically wrong outputs, such as PIDs instead of source IPs
- trusting Grafana UI state over Prometheus rule health
- replaying rotated or compressed mail history into the live ingestion path

## References

- [Mail Stack Boundary Design](../../lab/primitives/Mail-Stack%20Boundary%20Design.md)
- [Mail Log Archival](../../lab/primitives/Mail%20Log%20Archival.md)
- [Mail Observability Pipeline](../../lab/primitives/mail-observability-pipeline.md)
- [Grafana Alloy Migration for the Mail Host](../../lab/research/grafana-alloy-mail-migration.md)
- [Mail Observability Rollout](./2026-03-04-mail-observability-rollout.md)
