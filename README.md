# 🏠 Homelab

A self-managed, production-grade homelab running on repurposed hardware. Built to learn by doing — everything here has been broken, debugged, and rebuilt at least once.

---

## Hardware

| Device | Role |
|---|---|
| Repurposed PC | Primary server — runs Proxmox VE |
| Raspberry Pi (×2) | DNS — redundant Pi-hole instances |

---

## Architecture Overview

```

Internet
    │
    ▼
Cloudflare (DNS, WAF, Bot Protection, Proxied Records)
    │
    ▼
Home Router
    │
    ├──► Raspberry Pi #1 (Pi-hole) ─┐
    └──► Raspberry Pi #2 (Pi-hole) ─┘ ← Redundant DNS filtering
                                    │
                                    ▼
                            Proxmox VE Host
                         ┌──────────────────────────────────────┐
                         │                                      │
                         │  LXC: Caddy (Reverse Proxy / HTTPS) │◄── External traffic
                         │    ├── → Jellyfin LXC               │    (Jellyfin + Seer only)
                         │    └── → Jellyseerr LXC             │
                         │                                      │
                         │  VM: TrueNAS SCALE                   │
                         │    └── ZFS RAIDZ1 pool               │
                         │        ├── SMB/NFS shares            │
                         │        └── Scheduled snapshots       │
                         │                                      │
                         │  VM: Debian (Docker host)            │
                         │    └── Gluetun (VPN container)       │
                         │         ├── Prowlarr (indexers)      │
                         │         ├── Sonarr (TV)              │
                         │         ├── Radarr (Movies)          │
                         │         ├── qBittorrent              │
                         │         ├── Bazarr (Subtitles)       │
                         │         ├── FlareSolverr             │
                         │         └── all routed through VPN   │
                         └──────────────────────────────────────┘

```

> All Docker containers on the Debian VM route traffic through **Gluetun**, acting as a VPN gateway and kill switch. If the VPN drops, all container traffic is blocked.

---

## Services

### Proxmox VE — Virtualisation Host
Runs all VMs and LXC containers on repurposed hardware. Manages resource allocation, snapshots, and VM lifecycle.

### TrueNAS SCALE (VM)
- **Storage:** ZFS RAIDZ1 pool for redundancy and data integrity
- **Sharing:** SMB and NFS shares for network file access
- **Backups:** Scheduled ZFS snapshots for point-in-time recovery
- **Access control:** Role-based permissions with least-privilege user accounts

> **Incident:** Experienced a data corruption event in 2025. Diagnosed the failure, executed full recovery from ZFS snapshots, and restored all services with zero permanent data loss. Now maintain a stricter snapshot schedule as a result.

### Caddy (LXC) — Reverse Proxy
- Automatic HTTPS via Let's Encrypt
- Routes external traffic exclusively to Jellyfin and Jellyseerr
- Sits behind Cloudflare for WAF, bot protection, and DDoS mitigation

### Pi-hole (×2 Raspberry Pi) — DNS
- Network-wide ad and tracker blocking
- Redundant instances for high availability — if one Pi fails, DNS resolution continues uninterrupted

### Media Stack (Debian VM → Docker → Gluetun VPN)

| Container | Purpose |
|---|---|
| **Gluetun** | VPN gateway + kill switch for all containers |
| **Prowlarr** | Centralised indexer management for Sonarr & Radarr |
| **Sonarr** | TV show monitoring and automated downloads |
| **Radarr** | Movie monitoring and automated downloads |
| **qBittorrent** | Download client, traffic routed through Gluetun |
| **Bazarr** | Automated subtitle downloads for Sonarr/Radarr |
| **FlareSolverr** | Cloudflare bypass proxy for Prowlarr indexers |

**Request flow:**
```
Jellyseerr (user request)
    │
    ├──► Sonarr (TV) ──► Prowlarr ──► Indexers
    │         └──────────────────────► qBittorrent (via Gluetun VPN)
    │
    └──► Radarr (Movies) ──► Prowlarr ──► Indexers
              └──────────────────────────► qBittorrent (via Gluetun VPN)
```

### Jellyfin (LXC) — Media Server
Self-hosted media streaming with multi-user access and library management. Publicly accessible via Caddy + Cloudflare.

### Jellyseerr (LXC) — Request Management
User-facing request interface connected to both Sonarr and Radarr. Publicly accessible alongside Jellyfin.

---

## Security Practices

- **Cloudflare WAF** — all publicly exposed services sit behind Cloudflare with WAF rules and bot protection enabled
- **Minimal attack surface** — only Jellyfin and Jellyseerr are externally reachable; all other services are LAN-only
- **VPN kill switch** — Gluetun ensures no container traffic leaks if the VPN connection drops
- **Least-privilege access** — separate user accounts with scoped permissions across TrueNAS shares and Proxmox
- **RBAC** — role-based access control across all shared storage resources
- **Automatic HTTPS** — Caddy handles TLS certificate provisioning and renewal automatically

---

## Skills Demonstrated

`Proxmox VE` `Linux (Ubuntu Server / Debian)` `Docker` `Docker Compose` `TrueNAS SCALE` `ZFS` `Networking` `DNS` `Reverse Proxy` `Cloudflare` `RBAC` `Incident Response` `Security Hardening` `Self-hosting`
