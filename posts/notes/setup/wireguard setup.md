---
title: Minimal WireGuard setup notes
slug: minimal-wireguard-setup-notes
type: runbook
status: draft
date: undated
updated: 2026-04-17
tags:
  - setup
  - wireguard
  - vpn
summary: This note records a small personal WireGuard hub setup with examples for restricted-peer access and full-subnet access.
verification_status: partial
verified_on:
---

# Minimal WireGuard setup notes

## Context

These are notes from setting up a small WireGuard network for personal infrastructure.

This is not a full guide. It documents what worked, what did not, and what would change later.

## Goal

- use a single VPS as a small WireGuard hub
- give peers private service access
- avoid unnecessary site-to-site routing complexity

## Environment

- single VPS acting as a hub
- a few personal machines as peers
- no mesh or automation

## The Final State

### Server-side configuration for a restricted peer

Used when I only wanted a peer to access the server itself.

```ini
[Interface]
Address = 10.88.0.1/24
PrivateKey = <server private key>

[Peer]
PublicKey = <peer public key>
AllowedIPs = 10.88.0.2/32
PersistentKeepalive = 25
```

This limits the peer to a single IP and avoids accidental routing leaks.

### Server-side configuration for full subnet access

Used when I wanted the peer to reach other services behind the tunnel.

```ini
[Interface]
Address = 10.88.0.1/24
PrivateKey = <server private key>

[Peer]
PublicKey = <peer public key>
AllowedIPs = 10.88.0.0/24
PersistentKeepalive = 25
```

This trades isolation for convenience and requires more trust in the peer.

## Verification

Bring the interface up:

```bash
wg-quick up wg0
```

Expected proof:

- the interface comes up cleanly
- the peer can reach only the intended addresses for the chosen `AllowedIPs` model

## Failure Modes Worth Caring About

- accidentally broadening `AllowedIPs` and turning a restricted peer into a routed subnet peer
- treating convenience as isolation
- assuming the hub-only topology scales without later design cleanup
