# Troubleshooting Guide

## UI stuck on "Loading..."

**Most common cause:** Browser blocks background API/WebSocket calls due to self-signed certificate.

**Fix:**
1. Open `https://<CONTAINER-IP>:5000/api/health` directly in the browser
2. Accept the certificate warning
3. Go back to `https://<CONTAINER-IP>:5000`
4. Hard refresh: `Ctrl+F5`

## curl from Docker host fails

```
curl: (7) Failed to connect ... Could not connect to server
```

**Cause:** Expected macvlan behavior. The Docker host cannot reach macvlan containers by default.

**Fix:** Test from a LAN client (another PC/server in the same subnet).

## `Destination Host Unreachable` from host

Same as above. If you need host-to-container access, create a macvlan host interface (see `architecture.md`).

## Container not reachable from LAN

**Cause:** Wrong `parent=` interface in the macvlan network definition.

**Fix:**
```bash
# Check current parent
docker network inspect <NETWORK-NAME> --format '{{json .Options}}'

# Recreate with correct parent
docker compose down
docker network rm <NETWORK-NAME>
docker network create -d macvlan \
  --subnet=<SUBNET> \
  --gateway=<GATEWAY> \
  -o parent=<CORRECT-INTERFACE> \
  <NETWORK-NAME>
docker compose up -d
```

## Container unhealthy

```bash
docker inspect --format='{{json .State.Health}}' pegaprox
docker logs --tail=200 pegaprox
```

Common causes:
- Volume permission issues
- Config DB corrupted
- Port already in use

## ARP entry shows FAILED

```
172.22.222.103 dev ens18 FAILED
```

This is the host-to-macvlan isolation. Normal behavior. Test from LAN client.
