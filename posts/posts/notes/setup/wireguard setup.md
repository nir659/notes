# Notes on a minimal WireGuard setup

These are notes from setting up a small WireGuard network for personal infrastructure.
This is not a guide. It documents what worked, what didn’t, and what I’d change.

## Context

- Single VPS acting as a hub
- A few personal machines as peers
- Goal: private service access, not site-to-site routing
- No fancy mesh or automation

---

## Server-side configuration (restricted peer)

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

---

## Server-side configuration (full subnet access)

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

---

## Bringing the interface up
```bash
wg-quick up wg0
```