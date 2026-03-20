# Architecture Details

## macvlan Networking

macvlan creates a virtual network interface with its own MAC address on top of a physical interface.

### Key properties
- Container gets a **real LAN IP** (Layer 2)
- No NAT, no port mapping required
- Container is directly reachable from LAN clients
- **Docker host cannot reach macvlan containers by default** (Linux kernel restriction)

### Host-to-container workaround (optional)
If you need the Docker host to reach the container (e.g. for monitoring scripts):

```bash
# Replace ens18 with your parent interface
# Choose a free IP in your subnet (not the container IP)
sudo ip link add pegaprox-link link ens18 type macvlan mode bridge
sudo ip addr add 192.168.0.200/24 dev pegaprox-link
sudo ip link set pegaprox-link up
```

To make this persistent across reboots, add it to `/etc/network/interfaces` or a systemd unit.

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 5000 | HTTPS | Web UI + REST API |
| 5001 | WSS | VNC WebSocket proxy |
| 5002 | WSS | SSH WebSocket proxy |

## Volumes

| Volume | Mount | Content |
|--------|-------|---------|
| `pegaprox-config` | `/app/config` | SQLite DB, encryption keys, SSL certs |
| `pegaprox-logs` | `/app/logs` | Application logs |

## SSL / TLS

On first start, PegaProx generates a **self-signed certificate** stored in `/app/config/ssl/`.

To use your own certificate:
1. Place `cert.pem` and `key.pem` in the `pegaprox-config` volume under `ssl/`
2. Restart the container: `docker compose restart pegaprox`
