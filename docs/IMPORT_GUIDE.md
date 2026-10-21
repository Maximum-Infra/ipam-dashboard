# Bulk CSV import into NetBox

Lessons learned importing real site/subnet/IP data into NetBox in bulk,
generalized with synthetic data. Example CSVs matching this structure are
in [`../examples/`](../examples/), using the IANA/RFC 5737 documentation
ranges (`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`) so they're
guaranteed not to collide with anyone's real network.

## Recommended source workbook

Keep one spreadsheet as the human-readable master file, even though NetBox
imports each object type as its own CSV:

| Sheet | Purpose |
|---|---|
| `Sites` | One row per site |
| `VLANs` | VLAN group, VID, name, role, status per site |
| `Subnets` | Prefixes — the most important sheet |
| `Used_IPs` / `IP_Ranges` | Gateways and fixed/reserved addresses |
| `Notes` | Anything messy or unconfirmed — free text |

Minimum useful `Subnets` columns:

```
site_name, environment, subnet_cidr, vlan_id, vlan_name, gateway_ip,
purpose, role, status, owner, notes
```

## Import order

1. Sites
2. VLAN groups / VLANs
3. Parent **container** prefixes
4. Active subnets (prefixes)
5. Gateway IPs / reserved ranges
6. (Optional) individual IP addresses

Do a **small test import first** — 2–5 subnets and their gateways — and
check the result before importing everything.

## Quirks found along the way

**NetBox's own Import page is the source of truth for required headers.**
Before writing a CSV by hand, open `IPAM → Prefixes → Import` (or
`IP Addresses`, `IP Ranges`, `Sites`) and read the pre-filled header line —
required columns differ by NetBox version.

- **IP Ranges import rejects a `name` column.** The required headers are
  `start_address,end_address`. Put any human-readable label in
  `description`/`comments` instead.
- **Role lookups fail if the role doesn't already exist**, with an error
  like `Object not found: DHCP`. Either pre-create the role in NetBox
  first, or drop the `role` column entirely for the first import pass —
  roles can be bulk-assigned afterward.
- **IP Ranges need the CIDR mask on both the start and end address**, e.g.
  `192.0.2.10/24,192.0.2.20/24`, not bare IPs.
- **A prefix has exactly one optional VLAN field.** If a subnet is shared
  across multiple VLANs (e.g. behind a third-party gateway), leave `VLAN`
  blank on the prefix and list the VLANs in `description`/`comments`
  instead of creating duplicate prefixes.
- **Parent/container prefixes should usually be unscoped.** If a `/16` (or
  `/8`) container spans more than one site, don't set `Site`/`Scope` on it
  — only the real child subnets should be scoped to a site. A newer NetBox
  may also use a `scope_type` + `scope_id` pair instead of a plain `site`
  column for prefixes; check the Import page's pre-filled headers.
- **`/29`-and-smaller public/WAN blocks must be entered as the network
  address**, not a host IP inside it — e.g. a documented gateway of
  `203.0.113.1/29` belongs to the prefix `203.0.113.0/29`.

## Validation checklist after import

- `Organization → Sites` — no duplicate or near-duplicate names
- `IPAM → Prefixes` with `Max Depth: all` — everything nests under the
  right container block, no unexpected top-level entries
- Orphan `IP Addresses` or `IP Ranges` whose subnet doesn't appear in
  `Prefixes` — confirm whether they're real before keeping them
- Cross-check any address that falls **outside** its site's documented
  subnet range against the source data — likely a transcription typo
- Decide on one consistent model and stick to it: e.g. prefixes for
  subnets, IP Ranges for reserved/fixed blocks, and either skip individual
  `IP Address` objects or use them consistently — mixing models per site
  makes the eventual dashboard harder to build cleanly
