# IPAM Dashboard — Internal IP & Subnet Dashboard (Design)

A design and deployment write-up for an internal IP Address Management (IPAM)
dashboard: a single place to see which subnets and IPs are already in use
across an organization's production, lab, and office environments, and to get
a clean recommendation for the next available block before spinning up a new
site.

This repo documents the **concept, architecture, and NetBox deployment
process**. It intentionally does not include any real site names, IP ranges,
or device inventory — see [Notes on scope](#notes-on-scope) below.

## The problem

Growing a studio/lab footprint across multiple sites makes it easy to lose
track of which subnets are already claimed. The goal was a tool that lets
technicians:

- View all sites/studios/labs and the subnets assigned to each
- Search by site, environment, VLAN, subnet, or IP
- Open a subnet and see used / free / reserved / conflicting addresses
- Get a recommendation for the next free block (e.g. "use `10.40.30.0/24`
  for the next studio")
- Add new sites/subnets/IPs with permissions
- Export reports
- Optionally compare documented IPs against live network discovery later

The interface was designed to be colorful and "hi-tech" rather than a bare
admin table — see the dashboard concept below.

![IPAM dashboard concept](docs/dashboard-concept.png)

## Architecture

Rather than build subnet/IP database logic from scratch, the design uses
[NetBox](https://github.com/netbox-community/netbox) as the source-of-truth
backend (prefixes, IP addresses, IP ranges, VLANs, sites) and layers a custom
dashboard UI on top, talking only to NetBox's REST API.

```
 Users / Technicians
         |
         v
 Reverse Proxy (Nginx / Caddy / Traefik)
         |
   +-----+------+
   |            |
   v            v
 Custom      NetBox
 Dashboard   Web UI
 (React/     (admin)
 Next.js)
   |
   v
 Backend API / Middleware (FastAPI or Node.js)
   |
   v
 NetBox REST API
   |
   v
 PostgreSQL + Redis
```

**Why this split:**

- NetBox already models prefixes, IP addresses, IP ranges, VLANs, sites, and
  exposes them through a token-authenticated REST API — no need to reinvent
  subnet-hierarchy or address-tracking logic.
- The custom dashboard owns the look/feel and the "available subnet"
  recommendation logic, but never writes directly to NetBox's database —
  only through its API, so the data model stays safe and upgrade-friendly.
- Docker Compose keeps the whole stack (NetBox + Postgres + Redis + the
  dashboard) portable and simple to deploy on a single internal VM.

### Recommended stack

| Layer | Choice |
|---|---|
| Backend of record | NetBox (Docker) |
| Database | PostgreSQL + Redis (via NetBox) |
| Dashboard frontend | React / Next.js + TypeScript, Tailwind CSS |
| Dashboard backend | FastAPI or Node.js (talks to NetBox REST API only) |
| Reverse proxy | Nginx or Caddy |
| Auth | Local users first, LDAP/SSO later |

### Suggested infrastructure

A single internal VM is enough to start:

- Ubuntu Server 24.04 LTS
- 2–4 vCPU, 8 GB RAM, 80–150 GB SSD (2 vCPU / 4 GB / 50 GB is enough for an
  MVP)
- Static internal IP, internal-only DNS record
- Nightly backup to a separate location

## Deployment phases

1. **Core IPAM backend** — deploy NetBox, import real sites/subnets/IPs. You
   already have a working IPAM database before any custom UI exists.
2. **Custom dashboard MVP** — read-only dashboard on top of NetBox: subnets
   by site, used vs. free IPs, utilization, available-subnet
   recommendations, overlap alerts.
3. **Production hardening** — HTTPS, user roles, backups, audit log,
   documentation.
4. **Optional network discovery** — compare NetBox's documented state
   against DHCP leases / switch ARP tables / DNS, and surface mismatches as
   suggestions rather than auto-overwriting NetBox.

## Permissions model

A simple role ladder was designed for this:

- **Admin** — full access to NetBox and the dashboard
- **Network Owner** — add/edit sites, subnets, VLANs, IPs
- **Technician** — view everything, request/reserve IPs, cannot delete
  subnets
- **Read-only Viewer** — search and export only

With an optional approval flow for new subnet requests: technician requests
→ network owner approves → subnet goes active → dashboard logs the approval.

## NetBox setup

See [`docs/NETBOX_SETUP.md`](docs/NETBOX_SETUP.md) for the generic,
reusable steps used to stand up NetBox and build its initial data model
(sites, VLAN groups, VLANs, parent/container prefixes, subnets, roles).

## Bulk CSV import notes

See [`docs/IMPORT_GUIDE.md`](docs/IMPORT_GUIDE.md) for the import workflow
and the quirks found along the way (required headers per object type, role
lookups failing on first import, IP Range import rejecting a `name` column,
parent-prefix scoping, and a post-import validation checklist). Generic
example CSVs matching this structure are in [`examples/`](examples/).

## Status

The NetBox backend, data model, and import workflow described here were
built and validated against a real internal deployment. The custom
dashboard UI shown above is an approved design concept; it had not yet been
built when this write-up was prepared.

## Notes on scope

The real deployment this design came from manages IP space for several
actual client sites. None of those site names, real IP ranges, device
inventories, or credentials are included in this repository — only the
reusable design, architecture, and generic NetBox setup/import process,
with synthetic example data (`examples/`) using the IANA/RFC 5737
documentation ranges (`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`)
and private RFC 1918 space.

## License

MIT — see [LICENSE](LICENSE).
