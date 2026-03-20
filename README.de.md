# PegaProx auf Docker (macvlan) – Installation & Best Practices

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/Docker-macvlan-blue)](https://docs.docker.com/network/macvlan/)
[![Proxmox](https://img.shields.io/badge/Proxmox-VE-orange)](https://www.proxmox.com/)

> Community-Anleitung für den Betrieb von **PegaProx** als Docker-Container mit macvlan-Netzwerk in Proxmox VE Umgebungen.

---

## Was ist PegaProx?

PegaProx ist ein webbasiertes Management-Tool für **Proxmox VE**. Es bietet eine zentrale Oberfläche für:

- Verwaltung von Clustern und Nodes
- VM-/Container-Lifecycle-Aktionen (Start/Stop/Status, Operationen)
- Live-Updates und Event-Streams
- Web-Konsolen / Remote-Zugriff über WebSockets (VNC/SSH)

---

## Technische Architektur

```
┌─────────────────────────────────────────────────────────┐
│                   LAN / Subnetz                         │
│                z.B. 172.22.222.0/24                     │
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
│  │  Docker Host     │  ← kann macvlan-Container         │
│  │  (z.B. swarm03)  │    standardmäßig NICHT direkt     │
│  │  ens18 / vmbr0   │    erreichen (erwartetes          │
│  └──────────────────┘    macvlan-Verhalten)             │
└─────────────────────────────────────────────────────────┘
```

**Komponenten:**
| Komponente | Beschreibung |
|---|---|
| PegaProx Container | Web UI + API (HTTPS/5000), VNC WS (5001), SSH WS (5002) |
| Volume `pegaprox-config` | Konfiguration, DB, SSL-Zertifikate → `/app/config` |
| Volume `pegaprox-logs` | Anwendungs-Logs → `/app/logs` |
| macvlan-Netzwerk | Container bekommt eigene LAN-IP, direkter L2-Zugriff auf Proxmox-Nodes |
| Proxmox VE API | TCP/8006 auf jedem Node |

---

## Voraussetzungen

- Linux-Host mit **Docker** und **Docker Compose v2**
- Vorhandenes **externes macvlan-Netzwerk** in Docker
- Freie LAN-IP für den PegaProx-Container
- Vorhandene Docker Volumes: `pegaprox-config` und `pegaprox-logs`
- Proxmox-Nodes von der Container-IP aus auf TCP/8006 erreichbar

---

## Schnellstart

### 1. Volumes anlegen (falls noch nicht vorhanden)
```bash
docker volume create pegaprox-config
docker volume create pegaprox-logs
```

### 2. macvlan-Netzwerk anlegen (falls noch nicht vorhanden)
```bash
# ens18 durch dein echtes Parent-Interface ersetzen
# Subnetz/Gateway an dein LAN anpassen
docker network create -d macvlan \
  --subnet=192.168.0.0/24 \
  --gateway=192.168.0.1 \
  -o parent=ens18 \
  MEIN-MACVLAN-NETZWERK
```

### 3. Compose-Datei anpassen
Kopiere `compose/docker-compose.yml` und passe an:
- `TRUSTED_PROXIES` → dein LAN-Subnetz
- `ipv4_address` → freie LAN-IP für den Container
- `networks.pegaprox_macvlan.name` → Name deines macvlan-Netzwerks

### 4. Starten
```bash
mkdir -p /opt/pegaprox
cp compose/docker-compose.yml /opt/pegaprox/
cd /opt/pegaprox
docker compose up -d
docker logs --tail=100 pegaprox
```

### 5. Web UI öffnen
```
https://<CONTAINER-IP>:5000
```
Beim ersten Start das self-signed Zertifikat im Browser akzeptieren.

---

## Standard-Login

| Feld | Wert |
|---|---|
| Benutzername | `pegaprox` |
| Passwort | `admin` |

> ⚠️ **Passwort sofort nach dem ersten Login ändern!**

---

## Best Practices

- Dedizierter **Proxmox Service-User + API-Token** verwenden (niemals `root@pam`)
- Netzwerkzugriff auf benötigte Ports beschränken (5000/5001/5002 und 8006)
- Monitoring/Alerts in den PegaProx-Einstellungen aktivieren
- Eigenes TLS-Zertifikat nutzen (`cert.pem` + `key.pem` im Config-Volume unter `ssl/` ablegen)

---

## Troubleshooting

| Symptom | Ursache | Lösung |
|---|---|---|
| UI bleibt bei „Loading" | Browser blockiert self-signed Cert für WebSocket/SSE | `/api/health` öffnen, Zertifikat akzeptieren, Hard-Reload (Strg+F5) |
| `curl` vom Docker-Host schlägt fehl | Erwartetes macvlan-Verhalten | Von einem LAN-Client testen |
| `Destination Host Unreachable` vom Host | macvlan-Isolation | macvlan Host-Interface anlegen (siehe Doku) |
| Container vom LAN nicht erreichbar | Falsches Parent-Interface | macvlan-Netzwerk mit korrektem `parent=` neu anlegen |

---

## Dateistruktur

```
pegaprox_proxmox_container_mcvlan/
├── README.md                  ← Englische Dokumentation
├── README.de.md               ← Diese Datei (Deutsch)
├── LICENSE
├── compose/
│   └── docker-compose.yml     ← Fertiges Compose-Template
└── docs/
    ├── architecture.md        ← Detaillierte Architektur-Notizen
    ├── troubleshooting.md     ← Erweiterter Troubleshooting-Guide
    └── proxmox-api-token.md   ← Proxmox API-Token erstellen
```

---

## Links

- [PegaProx GitHub](https://github.com/pegaprox/pegaprox)
- [Proxmox VE API Dokumentation](https://pve.proxmox.com/pve-docs/api-viewer/)
- [Docker macvlan Netzwerk](https://docs.docker.com/network/macvlan/)

---

## Lizenz

MIT – siehe [LICENSE](LICENSE)
