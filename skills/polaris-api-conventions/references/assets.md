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
