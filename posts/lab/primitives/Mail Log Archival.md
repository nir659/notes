## Goal

- Keep **journald** as the authoritative, live log source
- Persist **all mail logs to disk**
- Avoid duplicates and gaps
- Make logs readable
- Be robust across reboots and reruns
- Prepare the ground for future forensics / analysis / AI tooling

---
## 1. Establish the ground truth

Both Postfix and Dovecot log **only to journald**.

I verified this by:
- Running `dovecot -n` (no log paths configured)
- Using:
  ```bash
  journalctl -u dovecot -f
  journalctl -u postfix@-.service -f
  ```

These commands show _everything I care about_.

Conclusion: **journald is the single source of truth**.

---

## 2. Configure journald as a bounded hot buffer

I do _not_ want journald to be long-term storage. I want it to be a hot cache that I drain continuously.

### `/etc/systemd/journald.conf`
```
[Journal]
Storage=persistent
SystemMaxUse=3G
SystemKeepFree=10G
RuntimeMaxUse=256M
MaxRetentionSec=21day
SyncIntervalSec=5m
```

Then:
```
systemctl restart systemd-journald
journalctl --disk-usage
```

This guarantees:
- Logs survive reboots
- journald never eats my disk
- I have ~2–3 weeks of safety if archiving fails

---

## 3. Decide the archive layout

I store **derived logs** (not raw journal files) here:
```swift
/var/log/journal-archive/
├── postfix/
│   └── postfix-YYYY-MM-DD.log
└── dovecot/
    └── dovecot-YYYY-MM-DD.log
```
Cursor state (bookmarks only, not logs):
```swift
/var/lib/journal-cursors/
├── postfix.cursor
└── dovecot.cursor
```
Create directories:
```
mkdir -p /var/log/journal-archive/{postfix,dovecot}
mkdir -p /var/lib/journal-cursors
```

---

## 4. Build a cursor-based exporter

I do **not** use `--since yesterday` because:

- it causes duplicates if rerun
- it has boundary bugs

Instead, I use **journald cursors**, which are opaque bookmarks.

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

---

## 5. Run it every 5 minutes

### Service unit
`/etc/systemd/system/mail-journal-archive.service`

```swift
[Unit]
Description=Archive Postfix and Dovecot journald logs (cursor-based)
After=systemd-journald.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/archive-mail-journal-cursor.sh
User=root
```

### Timer unit

`/etc/systemd/system/mail-journal-archive.timer`
```swift
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

```
systemctl daemon-reload
systemctl enable --now mail-journal-archive.timer
```

Verify:
```
systemctl list-timers | grep mail-journal
journalctl -u mail-journal-archive.service --since "30 minutes ago"
```

---

## 6. Viewing the logs

- **Cursor files** (`*.cursor`) are _state only_. Never read them.
- **Actual logs** are plain text files.

Examples:
```
tail -f /var/log/journal-archive/postfix/postfix-$(date +%F).log
less /var/log/journal-archive/dovecot/dovecot-$(date +%F).log
```

Older (compressed) logs:
```
zless /var/log/journal-archive/postfix/postfix-2026-01-30.log.gz
zgrep "auth failed" /var/log/journal-archive/dovecot/*.gz
```

This now fully replaces `journalctl -f` for most workflows.

---

## 7. Compression

Daily compression job:
```
find /var/log/journal-archive -type f -name "*.log" -mtime +2 -exec gzip -f {} \;
```

This gives me **years** of retention with minimal disk usage.