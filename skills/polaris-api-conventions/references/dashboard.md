# Dashboard & NOC feeds

_Generated from Polaris `public/api.html` (section `dashboard`). Do not edit by hand — re-run `scripts/import-api-html.mjs`._

Aggregate wallboard/kiosk feeds. All three endpoints follow the **filter-don't-403 contract**: sections the token's role cannot read come back *empty with the response shape unchanged*, never as an error — so a read-only kiosk token renders whatever it is entitled to. Responses are cached server-side for ~10 s and shared across callers.

### GET /dashboard/noc-summary

_Gate: per-feed: assets/events/alerts read_

The NOC feed bundle. `?feeds=` selects a comma-separated subset of `status, downNodes, downInterfaces, downIpsecTunnels, topCpu, topMemory, slowestResponse, packetLoss, diskUsage, temperature, storageForecast, stalePolls, sitesWithIssues, recentReboots, activeAlerts`; absent = all. Filters: `assetTypes`, `regionTags`, `fortigates` (all CSV), `limit` (≤1000; each feed has its own default cap), `samples` (top-N averaging depth, ≤100, default 10), `includeDependencyDown=1`.

Response is a flat map of the requested feeds. Three feeds fan out: `status` → `statusCounts` + `uptimePercent` + `activeAlertCount`; `downNodes` → `downNodes` + `downNodesTotal` (true uncapped count); `activeAlerts` → `activeAlerts` + `activeAlertsTotal`. Feed gates: `recentReboots` needs `events:read`, `activeAlerts` needs `alerts:read`, everything else `assets:read`.

```bash
curl -H "Authorization: Bearer $POLARIS_TOKEN" \
  "$POLARIS_URL/api/v1/dashboard/noc-summary?feeds=status,downNodes,activeAlerts&limit=50"
```

### GET /dashboard/summary

_Gate: per-section: ipBlocks/reservations/assets read_

`?sections=` subset of `blocks, recent, assetTypes, monitorAlerts` (absent = all). Response keys (only requested sections appear): `blockUtilization`, `recentReservations` (`recentSourceTypes` + `recentLimit` ≤1000 filter it), `assetTypeCounts`, and `monitorAlerts` + `monitorAlertsOverflow` (list capped at 50).

### GET /dashboard/filter-options

_Gate: assets:read_

`{ assetTypes, regions, fortigates }` — the values the noc-summary filters accept. Empty arrays (not 403) without `assets:read`.
