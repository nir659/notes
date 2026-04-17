---
title: Mail Log Archival
slug: mail-log-archival
type: runbook
status: draft
date: undated
updated: 2026-04-17
tags:
  - mail
  - journald
  - archive
  - postfix
  - dovecot
summary: Cursor-based journald export keeps mail archives readable, durable, and safe against replay or gap bugs.
verification_status: partial
verified_on:
---

# Mail Log Archival

## Context

This note defines how mail logs get exported from journald into durable plain-text archives.

The underlying rule is simple: journald is the authoritative live source, but it is not meant to be the only long-term store. The archive layer exists so history remains readable, compressible, and independent of Loki retention.

## Goal

- keep **journald** as the authoritative, live log source
- persist **all mail logs to disk**
- avoid duplicates and gaps
- make logs readable
- be robust across reboots and reruns
- prepare the ground for future forensics, analysis, and AI tooling

## Environment

- Postfix and Dovecot logging only to journald
- archive output under `/var/log/journal-archive/`
- cursor state under `/var/lib/journal-cursors/`
- systemd service and timer driving the exporter

## The Final State

Both Postfix and Dovecot log **only to journald**.

I verified this by:

- running `dovecot -n` with no log paths configured
- using:

```bash
journalctl -u dovecot -f
journalctl -u postfix@-.service -f
```

These commands show everything that matters, which confirms journald is the single source of truth.

### Bounded hot buffer

I do not want journald to be long-term storage. I want it to be a hot cache that I drain continuously.

`/etc/systemd/journald.conf`

```ini
[Journal]
Storage=persistent
SystemMaxUse=3G
SystemKeepFree=10G
RuntimeMaxUse=256M
MaxRetentionSec=21day
SyncIntervalSec=5m
```

Then:

```bash
systemctl restart systemd-journald
journalctl --disk-usage
```

This guarantees:

- logs survive reboots
- journald never eats the disk
- there is roughly 2 to 3 weeks of safety if archiving fails

### Archive layout

I store **derived logs** rather than raw journal files here:

```text
/var/log/journal-archive/
├── postfix/
│   └── postfix-YYYY-MM-DD.log
└── dovecot/
    └── dovecot-YYYY-MM-DD.log
```

Cursor state lives separately:

```text
/var/lib/journal-cursors/
├── postfix.cursor
└── dovecot.cursor
```

Create directories:

```bash
mkdir -p /var/log/journal-archive/{postfix,dovecot}
mkdir -p /var/lib/journal-cursors
```

### Cursor-based exporter

I do **not** use `--since yesterday` because:

- it causes duplicates if rerun
- it has boundary bugs

Instead, I use journald cursors, which are opaque bookmarks.

`/usr/local/bin/archive-mail-journal-cursor.sh`

```bash
#!/bin/bash
set -euo pipefail

BASE="/var/log/journal-archive"
STATE="/var/lib/journal-cursors"

mkdir -p "$STATE" "$BASE/postfix" "$BASE/dovecot"

append_unit() {
  local unit="$1"
  local name="$2"
  local cursor_file="$STATE/$name.cursor"
  local out_file="$BASE/$name/$name-$(date +%F).log"

  local cursor=""
  if [[ -s "$cursor_file" ]]; then
    cursor="$(head -n 1 "$cursor_file" | tr -d '\r\n')"
  fi

  local before_size after_size
  before_size=$(stat -c%s "$out_file" 2>/dev/null || echo 0)

  if [[ "$cursor" == s=* && "$cursor" == *";i="* ]]; then
    journalctl -u "$unit" --after-cursor="$cursor" \
      --no-pager --output=short-iso-precise >> "$out_file"
  else
    journalctl -u "$unit" --since "15 minutes ago" \
      --no-pager --output=short-iso-precise >> "$out_file"
  fi

  after_size=$(stat -c%s "$out_file" 2>/dev/null || echo 0)

  if (( after_size > before_size )); then
    local newest
    newest="$(journalctl -u "$unit" -n 1 --show-cursor --no-pager \
      | sed -n 's/^-- cursor: //p' | tail -n 1 | tr -d '\r\n')"

    if [[ "$newest" == s=* && "$newest" == *";i="* ]]; then
      echo "$newest" > "$cursor_file"
    fi
  fi
}

append_unit "postfix@-.service" "postfix"
append_unit "dovecot" "dovecot"
```

Make it executable:

```bash
chmod +x /usr/local/bin/archive-mail-journal-cursor.sh
```

This guarantees:

- no duplicates
- no gaps
- safe reruns
- cursor advances **only when new logs were written**

### Run every 5 minutes

Service unit:

`/etc/systemd/system/mail-journal-archive.service`

```ini
[Unit]
Description=Archive Postfix and Dovecot journald logs (cursor-based)
After=systemd-journald.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/archive-mail-journal-cursor.sh
User=root
```

Timer unit:

`/etc/systemd/system/mail-journal-archive.timer`

```ini
[Unit]
Description=Run mail journal archiver every 5 minutes

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min
Persistent=true
AccuracySec=30s

[Install]
WantedBy=timers.target
```

Enable it:

```bash
systemctl daemon-reload
systemctl enable --now mail-journal-archive.timer
```

## Verification

Verify the timer and recent service activity:

```bash
systemctl list-timers | grep mail-journal
journalctl -u mail-journal-archive.service --since "30 minutes ago"
```

View current logs:

```bash
tail -f /var/log/journal-archive/postfix/postfix-$(date +%F).log
less /var/log/journal-archive/dovecot/dovecot-$(date +%F).log
```

View older compressed logs:

```bash
zless /var/log/journal-archive/postfix/postfix-2026-01-30.log.gz
zgrep "auth failed" /var/log/journal-archive/dovecot/*.gz
```

This now fully replaces `journalctl -f` for most archive-review workflows.

### Compression

Daily compression job:

```bash
find /var/log/journal-archive -type f -name "*.log" -mtime +2 -exec gzip -f {} \;
```

This gives years of retention with minimal disk usage.

## Failure Modes Worth Caring About

- time-sliced export reintroducing duplicates or gaps
- cursor files getting corrupted or advanced incorrectly
- archive timer stopping silently
- archive files growing stale while journald still looks healthy
- treating cursor files as logs instead of state

## References

- [Mail Observability Pipeline](./mail-observability-pipeline.md)
- [Grafana Alloy Migration for the Mail Host](../research/grafana-alloy-mail-migration.md)
- [Mail Observability Rollout](../../log/raw/2026-03-04-mail-observability-rollout.md)
- [From Pipes to Signal: Building a Mail Observability System](../../log/raw/2026-03-05-mail-observability-from-pipes-to-signal.md)
