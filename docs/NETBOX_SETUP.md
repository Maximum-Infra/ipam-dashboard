# NetBox setup

Generic, reusable steps for standing up NetBox as the IPAM backend and
building its initial data model. All names, IPs, and VLAN IDs below are
placeholder examples — replace them with your own.

## 1. Create the VM

- OS: Ubuntu Server 24.04 LTS
- Static internal IP, e.g. `10.x.x.x`
- Hostname: `ipam-prod-01`
- Internal DNS record, e.g. `ipam.example.local`

Install the basics and enable the firewall:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl ca-certificates git vim ufw

sudo ufw allow ssh
sudo ufw allow 443/tcp
sudo ufw enable

# Only while testing — close this once the HTTPS reverse proxy is working
sudo ufw allow 8000/tcp
```

## 2. Install Docker Engine + Compose

Use Docker's official apt repository rather than the distro's `docker.io`
package:

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

docker --version
docker compose version

# optional: run docker without sudo (grants host-level trust — admins only)
sudo usermod -aG docker $USER
```

## 3. Deploy NetBox with Docker

Uses the official [netbox-community/netbox-docker](https://github.com/netbox-community/netbox-docker)
project:

```bash
cd /opt
sudo git clone -b release https://github.com/netbox-community/netbox-docker.git
sudo chown -R $USER:$USER /opt/netbox-docker
cd /opt/netbox-docker
cp docker-compose.override.yml.example docker-compose.override.yml
docker compose pull
docker compose up -d

docker compose ps
docker compose logs -f netbox
```

Open `http://<server-ip>:8000` temporarily, then create the first admin
user:

```bash
docker compose exec netbox /opt/netbox/netbox/manage.py createsuperuser
```

Put NetBox behind a reverse proxy (Nginx/Caddy/Traefik) with HTTPS before
exposing it beyond the local network, and close port 8000 to anything but
localhost/admin once that's working.

## 4. Build the initial data model

NetBox's object model, in the order that works best for import:

1. **Sites / Locations**
2. **VLAN Groups**
3. **VLANs**
4. **Parent / container prefixes**
5. **Real subnets (prefixes)**
6. **Gateway IPs**
7. **Tags / Prefix-VLAN roles** (optional — can be skipped for a first pass)

### 4.1 Create a Site

`Organization → Sites → Add`

```
Name: Site A
Slug: site-a
Status: Active
Description: Example site
```

### 4.2 Create a VLAN Group

`IPAM → VLAN Groups → Add`

```
Name: Site A VLANs
Scope type: DCIM > Site
Scope: Site A
Slug: site-a-vlans
```

Scoping the VLAN group to the site keeps VLAN IDs tied to that site instead
of being global.

### 4.3 Create starter VLANs

`IPAM → VLANs → Add` — only create a few to start:

```
VID 10  Management   Group: Site A VLANs   Role: Management
VID 20  Production   Group: Site A VLANs   Role: Production
VID 30  Lab          Group: Site A VLANs   Role: Lab
```

### 4.4 Create parent (container) blocks

Create these **before** adding small subnets, so NetBox can show hierarchy
and free space later.

`IPAM → Prefixes → Add`

```
Prefix: 10.10.0.0/16   Status: Container   Role: Production   Description: Production parent block
Prefix: 10.20.0.0/16   Status: Container   Role: Lab          Description: Lab parent block
Prefix: 10.30.0.0/16   Status: Container   Role: Office        Description: Office parent block
```

**Important:** if a parent block is shared across more than one site, leave
its `Scope`/`Site` blank. Scoping a shared `/16` to a single site is
misleading — scope only the real, active child subnets to their actual
site.

### 4.5 Create a real subnet

```
Prefix: 10.10.20.0/24
Status: Active
Site: Site A
VLAN: 20 (Production)
Role: Production
Description: First production subnet
```

It should appear nested under its `/16` parent in the Prefixes list.

### 4.6 Add the gateway IP

`IPAM → IP Addresses → Add`

```
Address: 10.10.20.1/24
Status: Active
DNS Name: gateway-site-a
Description: Default gateway for 10.10.20.0/24
```

Add just the gateway and a handful of important static devices by hand —
bulk import (see [`IMPORT_GUIDE.md`](IMPORT_GUIDE.md)) handles the rest.

## 5. A note on VLAN-to-prefix modeling

A NetBox prefix has **one** optional VLAN field — one subnet maps to one
Layer-2 domain. If the same subnet is genuinely shared across several
VLANs (e.g. behind a third-party gateway), don't create duplicate prefixes
for each VLAN in the same VRF. Instead:

- Create the prefix once, leave `VLAN` blank, and list the VLANs it serves
  in the description/comments, **or**
- If the VLANs are truly separate, isolated routing domains that happen to
  reuse the same address space, model each one with its own VRF.

## 6. Validate before building the dashboard

Before moving on to the dashboard UI, check:

- `Organization → Sites` — no duplicate/near-duplicate site names
- `IPAM → Prefixes` with `Max Depth: all` — private ranges nest correctly
  under their container blocks
- `IPAM → IP Ranges` — ranges look logical per site
- Any orphan IP addresses/prefixes that don't belong to an expected parent
  — confirm before keeping them

See [`IMPORT_GUIDE.md`](IMPORT_GUIDE.md) for the full CSV import workflow
and validation checklist.
