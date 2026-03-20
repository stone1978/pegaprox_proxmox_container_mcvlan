# Creating a Proxmox API Token for PegaProx

It is strongly recommended to use a **dedicated service user with an API token** instead of `root@pam`.

## Step 1: Create a dedicated user (Proxmox GUI)

1. Go to **Datacenter → Users → Add**
2. User ID: `pegaprox-api`
3. Realm: `pve` (Proxmox VE authentication)
4. Set a strong password

## Step 2: Assign permissions

1. Go to **Datacenter → Permissions → Add → User Permission**
2. Path: `/`
3. User: `pegaprox-api@pve`
4. Role: `Administrator` (or `PVEAdmin` for slightly reduced privileges)
5. Check **Propagate**

## Step 3: Create an API token

1. Go to **Datacenter → API Tokens → Add**
2. User: `pegaprox-api@pve`
3. Token ID: `pegaprox`
4. **Uncheck** "Privilege Separation" (so the token inherits user permissions)
5. Click **Add** and **copy the token secret** – it is shown only once!

You will get:
- Token ID: `pegaprox-api@pve!pegaprox`
- Token Secret: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

## Step 4: Add cluster in PegaProx

1. Open PegaProx → **Add Cluster**
2. Enter the IP of one Proxmox node (e.g. `172.22.222.10`)
3. Authentication: **API Token**
4. Token ID: `pegaprox-api@pve!pegaprox`
5. Token Secret: `<your secret>`
6. PegaProx will auto-discover all other nodes in the cluster

## CLI alternative (on a Proxmox node)

```bash
# Create user
pveum user add pegaprox-api@pve --password <STRONG-PASSWORD>

# Assign role
pveum acl modify / --user pegaprox-api@pve --role Administrator

# Create API token (no privilege separation)
pveum user token add pegaprox-api@pve pegaprox --privsep 0
```
