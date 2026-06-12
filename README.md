# Network Health Manager — Infrastructure & Deployment

A self-hosted network diagnostics tool (ping, port scan, DNS lookup, and local-LLM analysis) running on a homelab Proxmox VM and exposed to the internet **without a public IP** via a Cloudflare Tunnel. This repository documents the full infrastructure, deployment, and security-hardening process end to end.

> This repo is **documentation only** — it captures how the system was built, deployed, and secured. Application source is kept in a separate repository.

---

## What this demonstrates

- **Virtualization & GPU passthrough** — Proxmox VM with a GTX 1650 passed through to the guest for CUDA-accelerated local LLM inference (Ollama), including the IOMMU / `vfio-pci` / Error 43 gotchas that come with it.
- **Networking around CGNAT** — no public IP and no port forwarding possible, solved with an outbound-only Cloudflare Tunnel.
- **Reverse-proxy architecture** — Nginx serving a static React build and proxying an `/api/` namespace to a FastAPI/Uvicorn backend.
- **TLS the hard way** — Let's Encrypt via DNS-01 challenge (port 80 is unreachable under CGNAT), automated renewal, Cloudflare set to Full (strict).
- **Defense in depth** — UFW, fail2ban (SSH + Nginx jails), Nginx rate-limiting zones, real-IP resolution behind the tunnel, a Cloudflare-only IP whitelist, and edge authentication via Cloudflare Access.
- **Operational rigor** — systemd service hardening, a verification checklist at every phase, and a documented troubleshooting log of real failures and fixes.

## Architecture

```
Browser
   │
   ▼
Cloudflare Edge ── Cloudflare Access (auth) ── cloudflared tunnel
                                                     │
                                                     ▼
                                            Nginx :80 / :443
                                              ├── /        → /var/www/<domain>/  (React static build)
                                              └── /api/    → Uvicorn :8000 (FastAPI)
                                                                  └── Ollama :11434 (locked to 127.0.0.1)
```

## Stack

| Component            | Role                                              |
| -------------------- | ------------------------------------------------- |
| Proxmox + Ubuntu 24.04 | Hypervisor and guest OS (GPU passthrough)       |
| Nginx                | Reverse proxy + static file server                |
| cloudflared          | Outbound tunnel to Cloudflare (CGNAT bypass)      |
| Cloudflare Access    | Edge authentication on the public hostname        |
| Certbot (snap)       | SSL via DNS-01 challenge + auto-renewal           |
| UFW + fail2ban       | Host firewall and brute-force/abuse protection    |
| FastAPI / Uvicorn    | Python API backend, port 8000                     |
| Ollama               | Local LLM inference, locked to 127.0.0.1:11434    |
| React + Vite         | Frontend, built to the Nginx web root             |

## Repository contents

```
docs/
  01-environment-setup.md   Proxmox VM creation, GPU passthrough, NVIDIA/CUDA, networking
  02-server-setup.md        Domain/Cloudflare, Nginx, tunnel, SSL, hardening, app migration, auth
  03-fail2ban.md            fail2ban concepts and SSH/Nginx jail configuration
config/
  nginx/site.conf.example         Hardened Nginx reverse-proxy config
  fail2ban/jail.local.example     SSH + Nginx jail overrides
  systemd/nhm.service.example     Hardened systemd unit for the backend
```

## Current state

| Item | Status |
| --- | --- |
| Proxmox VM + GPU passthrough | Working (`nvidia-smi` + CUDA verified) |
| Cloudflare Tunnel | Healthy — routing to the origin |
| TLS | Let's Encrypt (ECDSA), auto-renewal verified, Cloudflare Full (strict) |
| Hardening | UFW + fail2ban (3 jails) + Nginx rate limiting active |
| Authentication | Active — Cloudflare Access policy on the public hostname |
| Reboot resilience | All services restart automatically on boot |

## Notes on this documentation

The live domain, internal IPs, and usernames have been replaced with placeholders (`nethealth.example.com`, `192.168.x.x`, `youruser`). No tokens, certificates, or secrets are included — those paths are referenced but their contents are never committed.

---
