# PegaProx on Docker (macvlan) – Install & Best Practices

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/Docker-macvlan-blue)](https://docs.docker.com/network/macvlan/)
[![Proxmox](https://img.shields.io/badge/Proxmox-VE-orange)](https://www.proxmox.com/)

> Community guide for running **PegaProx** in a Docker container with macvlan networking on Proxmox VE environments.

---

## What is PegaProx?

PegaProx is a web-based management tool for **Proxmox VE**. It provides a central UI for:

- Managing clusters and nodes
- VM/CT lifecycle actions (start/stop/status, operations)
- Live updates and event streams
- Web consoles / remote access via WebSockets (VNC/SSH)

---

## Technical Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     LAN / Subnet                        │
│                  e.g. 172.22.222.0/24                   │
│                                                         │
│  ┌──────────────────┐      ┌──────────────────────────┐ │
│  │  PegaProx        │      │  Proxmox VE Nodes        │ │
│  │  Docker macvlan  │─────▶│  TCP/8006 (API)          │ │
│  │  172.22.222.103  │      │  172.22.222.x            │ │
│  │  HTTPS :5000     │      └──────────────────────────┘ │
│  │  VNC WS :5001    │                                   │
│  │  SSH WS :5002    │                                   │
│  └──────────────────┘                                   │
│                                                         │
│  ┌──────────────────┐                                   │
│  │  Docker Host     │  ← cannot reach macvlan           │
│  │  (swarm03)       │    container directly             │
│  │  ens18 / vmbr0   │    (expected macvlan behavior)    │
│  └──────────────────┘                                   │
└─────────────────────────────────────────────────────────┘
```

**Components:**
| Component | Description |
|---|---|
| PegaProx Container | Web UI + API (HTTPS/5000), VNC WS (5001), SSH WS (5002) |
| `pegaprox-config` volume | Config, DB, SSL certs → `/app/config` |
| `pegaprox-logs` volume | Application logs → `/app/logs` |
| macvlan network | Container gets own LAN IP, direct L2 access to Proxmox nodes |
| Proxmox VE API | TCP/8006 on each node |

---

## Prerequisites

- Linux host with **Docker** and **Docker Compose v2**
- Existing **external macvlan network** in Docker
- Free LAN IP for the PegaProx container
- Existing Docker volumes: `pegaprox-config` and `pegaprox-logs`
- Proxmox nodes reachable from the container IP on TCP/8006

---

## Quick Start

### 1. Create volumes (if not yet done)
```bash
docker volume create pegaprox-config
docker volume create pegaprox-logs
```

### 2. Create macvlan network (if not yet done)
```bash
# Replace ens18 with your actual parent interface
# Replace subnet/gateway with your LAN values
docker network create -d macvlan \
  --subnet=192.168.0.0/24 \
  --gateway=192.168.0.1 \
  -o parent=ens18 \
  MY-MACVLAN-NETWORK
```

### 3. Edit the Compose file
Copy `compose/docker-compose.yml` and edit:
- `TRUSTED_PROXIES` → your LAN subnet
- `ipv4_address` → free LAN IP for the container
- `networks.pegaprox_macvlan.name` → your macvlan network name

### 4. Start
```bash
mkdir -p /opt/pegaprox
cp compose/docker-compose.yml /opt/pegaprox/
cd /opt/pegaprox
docker compose up -d
docker logs --tail=100 pegaprox
```

### 5. Open UI
```
https://<CONTAINER-IP>:5000
```
Accept the self-signed certificate on first start.

---

## Default Login

| Field | Value |
|---|---|
| Username | `pegaprox` |
| Password | `admin` |

> ⚠️ **Change the password immediately after first login!**

---

## Best Practices

- Use a dedicated **Proxmox service user + API token** (never `root@pam`)
- Restrict network access to required ports only (5000/5001/5002 and 8006)
- Enable monitoring/alerts in PegaProx settings
- Use a proper TLS certificate (place `cert.pem` + `key.pem` in the config volume under `ssl/`)

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| UI stuck on "Loading" | Browser blocks self-signed cert for WebSocket/SSE | Open `/api/health`, accept cert, hard refresh (Ctrl+F5) |
| `curl` from Docker host fails | Expected macvlan behavior | Test from a LAN client instead |
| `Destination Host Unreachable` from host | macvlan isolation | Add a macvlan host interface (see docs) |
| Container not reachable from LAN | Wrong parent interface | Recreate macvlan network with correct `parent=` |

---

## File Structure

```
pegaprox_proxmox_container_mcvlan/
├── README.md                  ← This file (English)
├── README.de.md               ← German documentation
├── LICENSE
├── compose/
│   └── docker-compose.yml     ← Ready-to-use Compose template
└── docs/
    ├── architecture.md        ← Detailed architecture notes
    ├── troubleshooting.md     ← Extended troubleshooting guide
    └── proxmox-api-token.md   ← How to create a Proxmox API token
```

---

## Links

- [PegaProx GitHub](https://github.com/pegaprox/pegaprox)
- [Proxmox VE API Docs](https://pve.proxmox.com/pve-docs/api-viewer/)
- [Docker macvlan networking](https://docs.docker.com/network/macvlan/)

---

## License

MIT – see [LICENSE](LICENSE)
