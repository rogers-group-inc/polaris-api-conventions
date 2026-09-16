# Assets & inventory

_Generated from Polaris `public/api.html` (section `assets`). Do not edit by hand — re-run `scripts/import-api-html.mjs`._

Device inventory reads. Gate: `assets:read` unless noted.

### GET /assets

_Gate: assets:read_

List devices. Response: `{ assets, total, limit, offset }` — `total` is the full filtered count, so page with it rather than `assets.length`.

Query parameters:

- `limit` (default 50, max 10000), `offset` (default 0). There is no `page` parameter.
- `search` — case-insensitive match across hostname, DNS name, IP, MAC, asset tag, and assigned-to.
- `status`, `assetType`, `monitor` — comma-separated multi-value. `monitor` accepts `Unmonitored, Monitored, Dep. Down, Up, Missed, Warning, Down, Recovering, Passive, Pending`.
- Per-column text filters: pass the column name (`hostname`, `ipAddress`, `serialNumber`, `assetTag`, `manufacturer`, `model`, `os`, `macAddress`, `assignedTo`, `purchaseOrder`, `dnsName`, `description`) plus an optional `Op` of `contains` (default), `not_contains`, `empty`, `is_not_empty`.
- `department` (contains match), `lastSeenFrom` / `lastSeenTo` (`YYYY-MM-DD`).
- `sortBy` (whitelisted column names; an unknown value is a `400`) + `sortDir` (`asc` default when sorting; default order without `sortBy` is newest first).

```bash
curl -H "Authorization: Bearer $POLARIS_TOKEN" \
  "$POLARIS_URL/api/v1/assets?search=fw-&status=active&limit=25"
```

### GET /assets/:id

_Gate: assets:read_

One device — the full asset record at the top level (not wrapped), including monitoring state, discovered attributes, associated IPs and MAC addresses. `404` when absent.

### GET /assets/:id/sources

_Gate: assets:read_

Which discovery sources claim this device (Entra / Intune / AD / FortiGate / vCenter / agent / manual …). Response: a JSON array of `{ id, sourceKind, externalId, integration, observed, firstSeen, lastSeen }`.

### GET /assets/:id/sightings

_Gate: assetsQuarantine:read_

FortiGate DHCP/endpoint sighting history — where the device was last seen on the network. Response: a JSON array, newest first, each row carrying the FortiGate, source, IP, timestamps, plus resolved `subnetName` and `vlan`. Note the gate: this read belongs to the quarantine key, not `assets`.

### GET /assets/:id/dependencies

_Gate: assets:read_

The device's dependency context: `{ asset, effectiveParents, computedParents, overrideParents, hasOverride, children, childrenTruncated, childCount, haPeer }`. `children` lists infrastructure-type children only, capped at 300 (`childrenTruncated` flags overflow).

Creating a device and putting it under monitoring is **two calls**: `POST /assets` carries inventory fields only, and the monitoring switch plus its credential wiring live on `PUT /assets/:id`. Credentials themselves are *not* created over the API in the normal flow — an operator saves them once under Server Settings → Credentials and the integration references the stored row by id.

### POST /assets

_Gate: assets:write_

Create a device. Body (all optional, but give it at least an address or a hostname): `{ ipAddress?, macAddress?, hostname?, dnsName?, assetTag?, serialNumber?, manufacturer?, model?, assetType?, status?, location?, latitude?/longitude?, department?, assignedTo?, os?, acquiredAt?, warrantyExpiry?, purchaseOrder?, notes?, description?, tags? }`. `assetType` defaults to `other` and `status` to `active`. `201` with the created record.

**Monitoring fields are not accepted here** — `monitored` and every `*CredentialId` belong to the PUT below. A device created by this endpoint is a manual-source asset: only response time gets a source default (ICMP), so every other stream stays dark until a polling method is chosen for it.

```bash
curl -X POST -H "Authorization: Bearer $POLARIS_TOKEN"   -H "Content-Type: application/json"   -d '{"hostname":"sw-branch-12","ipAddress":"10.20.30.4","assetType":"switch"}'   "$POLARIS_URL/api/v1/assets"
```

### PUT /assets/:id

_Gate: assets:write_

Update any field `POST /assets` accepts, plus the monitoring surface: `{ monitored?, monitorCredentialId?, monitorIntervalSec? (5–86400), probeTimeoutMs? (100–60000) }`, the per-stream polling methods `{ responseTimePolling?, cpuMemoryPolling?, temperaturePolling?, interfacesPolling?, lldpPolling?, storagePolling? }` (one of `rest_api, snmp, winrm, ssh, icmp, agent, vcenter, fortimanager, disabled`), and the per-stream credentials `{ responseTimeCredentialId?, cpuMemoryCredentialId?, temperatureCredentialId?, interfacesCredentialId?, lldpCredentialId? }`. `null` clears a slot and falls back to the next settings tier.

Refusals worth coding against:

- `400` — `Credential … not found`, naming the id. It is well-formed but matches no stored credential (commonly one deleted and recreated in the UI, which issues a new id).
- `400` — a credential whose **type** cannot serve the stream's transport, e.g. an `ssh` credential on a stream polling over SNMP. The pairing is `snmp→snmp`, `winrm→winrm`, `ssh→ssh`, `rest_api→restapi`; ICMP, Disabled, Agent, vCenter and FortiManager take no per-asset credential, and a credential left beside one of those is accepted as staged config. `monitorCredentialId` is never type-checked — it is the fallback for every stream at once, and those streams may legitimately poll over different transports.
- `400` — a polling method the asset's discovery source does not support, or ICMP on any stream but response time.
- `409` — `monitored: true` on a status that cannot carry monitoring (decommissioned / disposed / lost / disabled). Change status first.
- `400` — setting or clearing `status: "quarantined"` (use the quarantine endpoints below).

```bash
curl -X PUT -H "Authorization: Bearer $POLARIS_TOKEN"   -H "Content-Type: application/json"   -d '{"monitored":true,"cpuMemoryPolling":"snmp","cpuMemoryCredentialId":"'$CRED_ID'"}'   "$POLARIS_URL/api/v1/assets/$ASSET_ID"
```

### POST /assets/bulk-monitor

_Gate: assets:write_

Flip monitoring on many devices at once: `{ ids: [uuid, …], monitored, monitorCredentialId?, monitorIntervalSec?, probeTimeoutMs? }` → `{ updated, errors: [{ id, error }] }`. A mixed selection is normal, so per-id problems (an unknown id, a status that cannot be monitored) are **reported in `errors` with HTTP 200** while the rest of the batch still applies. An unknown `monitorCredentialId` is the exception and fails the whole call with `400` — it applies to every id in the batch, so nothing would be written correctly.

### DELETE /assets/:id

_Gate: assets:write_

`204`. `409` for a quarantined device — release it first.

### GET /credentials

_Gate: credentials:read_

List stored monitoring credentials so an integration can resolve the id it needs by name: a JSON array of `{ id, name, type, config, createdBy, createdAt }` with **every secret masked** (community strings, passwords, private keys, passphrases and API tokens all read `••••••••`). `type` is one of `snmp, winrm, ssh, restapi, http`.
