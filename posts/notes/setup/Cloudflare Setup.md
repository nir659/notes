---
title: Cloudflare edge filtering notes
slug: cloudflare-edge-filtering-notes
type: runbook
status: draft
date: undated
updated: 2026-04-17
tags:
  - setup
  - cloudflare
  - edge
  - waf
summary: This note records a broad Cloudflare filtering rule and a separate ASN block rule used to drop noisy or obviously unwanted traffic at the edge.
verification_status: unverified
verified_on:
---

# Cloudflare edge filtering notes

## Context

This note records a Cloudflare filtering setup intended to block obviously low-value or malicious traffic before it reaches the origin.

## Goal

- block unwanted HTTP traffic and obviously suspicious clients at the edge
- reduce noise from bots, Tor, spoofed IPs, and odd request patterns
- keep a reusable record of the exact Cloudflare expression in use

## Environment

- Cloudflare edge filtering / WAF expressions
- rule set focused on HTTP request filtering and ASN blocking

## The Final State

The primary rule is intended to block:

1. HTTP connections that should not be allowed
2. a large amount of malicious user agents used for DDoS or crawling
3. anything above threat level 1
4. spoofed IPs
5. unsupported or suspicious HTTP versions
6. Tor
7. unknown states
8. known bots
9. requests that are not `GET` or `POST`
10. suspicious referers containing common low-trust domains
11. public TLS methods often associated with abusive traffic

## Implementation

### Primary filtering expression

```text
(cf.client.bot) or (http.user_agent contains "Cyotek") or (http.user_agent contains "python") or (http.user_agent contains "undefined") or (http.user_agent eq "Empty user agent") or (http.user_agent contains "HTTrack") or (http.user_agent contains "Java") or (http.user_agent contains "curl") or (http.user_agent contains "RestSharp") or (http.user_agent contains "Ruby") or (http.user_agent contains "Nmap") or (http.user_agent eq "libwww") or (not http.request.version in {"HTTP/1.0" "HTTP/1.1" "HTTP/1.2" "HTTP/2" "HTTP/3"}) or (ip.geoip.country eq "T1") or (ip.geoip.country eq "XX") or (cf.threat_score ge 1) or (not http.request.method in {"GET" "POST"}) or (http.user_agent contains " Uptime-Kuma") or (http.user_agent contains "sitechecker") or (http.user_agent contains "axios") or (http.referer contains "fbi.com") or (http.user_agent eq " Stripe/1.0 (+https://stripe.com/docs/webhooks)") or (http.cookie eq "cf_rate=mjRCQSuBeJk3Db") or (http.user_agent contains "Python") or (http.user_agent contains "YaBrowser") or (http.user_agent contains "lt-GtkLauncher") or (http.user_agent contains "3gpp-gba") or (http.user_agent contains "UNTRUSTED") or (http.user_agent contains "Galeon/1.3.15") or (http.user_agent contains "Phoenix/0.2" and http.user_agent contains "Fennec/2.0.1") or (http.user_agent contains "Minefield") or (http.user_agent contains ".NET") or (http.user_agent contains " Arora/0.8.0") or (http.referer eq "https://check-host.net") or (http.user_agent contains "https://check-host.net") or (http.user_agent contains "CheckHost") or (ip.src in {0.0.0.0/8 10.0.0.0/8 100.64.0.0/10 127.0.0.0/8 127.0.53.53 169.254.0.0/16 172.16.0.0/12 192.0.0.0/24 192.0.2.0/24 192.168.0.0/16 198.18.0.0/15 198.51.100.0/24 203.0.113.0/24 224.0.0.0/4 240.0.0.0/4 255.255.255.255/32}) or (http.user_agent eq "-") or (http.user_agent eq " ") or (http.user_agent eq "nginx-ssl early hints")
```

### ASN blocking

```text
(ip.geoip.asnum in {16276})
```

Source for ASN list:

- https://github.com/NullifiedCode/ASN-Lists

## Verification

This note currently stores the rule definitions themselves. It does not yet include Cloudflare event samples, before-and-after request stats, or explicit false-positive review data.

## Failure Modes Worth Caring About

- the expression becoming broad enough to block legitimate clients
- referer and user-agent matching creating noisy false positives
- bot or ASN rules dropping monitoring traffic unintentionally
- storing the expression without storing the operational evidence that it is safe
