# Path Monitor

_Generated from Polaris `public/api.html` (section `path-monitor`). Do not edit by hand — re-run `scripts/import-api-html.mjs`._

A **path check** is an HTTP / HTTPS request, a TCP connect or a ping that the **Polaris Agent** (0.21.0 or later) runs from each host it is pointed at, with an optional traceroute of the path. A check's hosts are its Sources filter plus any pinned assets, *intersected with hosts running an active agent*. A check has no threshold of its own and never changes a host's Up / Down status: alert on it with an automation on the `path*` metrics (`pathOk`, `pathLatencyMs`, `pathFailurePct`, `pathHttpStatus`, `pathHopCount`, `pathTlsDaysLeft`) or on the `path_check.path_changed` event. Check endpoints gate on the `pathChecks` key; the per-host readings under `/assets/:id` gate on `assets:read`, because they describe that asset.

### GET /path-checks

_Gate: pathChecks:read_

`{ checks: [...] }`, by name. Each check carries its definition (below) plus a fleet summary: `sourceCount` (hosts running it), `okCount` / `failCount` (hosts whose latest run passed / failed — a host with no result yet is in neither) and `lastRunAt`.

```bash
curl -H "Authorization: Bearer $POLARIS_TOKEN" \
  "$POLARIS_URL/api/v1/path-checks"
```

### GET /path-checks/:id

_Gate: pathChecks:read_

One check, same shape as a list row. `404` when it does not exist.

### GET /path-checks/:id/results

_Gate: pathChecks:read_

`{ results: [...] }` — every host running the check with its latest result, by hostname: `assetId, hostname, ipAddress, os, explicit` (pinned rather than matched), `lastOk, lastSampleAt, lastLatencyMs, lastHttpStatus, lastError, lastResolvedIp, lastFailAt, lastHopCount, lastTracerouteComplete, lastTracerouteAt`, and the agent's `agentVersion, online, supported` (`false` below 0.21.0 — the host is a member but cannot run it).

### POST /path-checks

_Gate: pathChecks:write_

Create a check; `201` with the stored check. Body:

- `name` (required, ≤120, unique — `409` on a duplicate), `description?`, `enabled?` (default `true`).
- `kind`: `http` | `https` | `tcp` | `icmp`. `target`: a full URL for HTTP / HTTPS (no credentials in it), `host:port` for TCP, a bare host for ICMP. IPv4 only. Refused with `400`: loopback, link-local, cloud-metadata and multicast addresses, and this Polaris server itself.
- `intervalSec?` — whole minutes, 60–3600, default 60. `timeoutMs?` — 500–30000, default 5000, and at most half the interval.
- `http?` (HTTP / HTTPS only): `{ expectStatus?` — e.g. `"200,204,300-399"`, empty = any 2xx; `bodyMatch?: { mode: "contains" | "regex" | "exact", pattern (≤1024), caseSensitive? }`; `verifyTls?` (default `true`) `}`. A regex must be RE2-compatible (no look-around or back-references) — the agent runs it.
- `traceroute?: { enabled?` (default `true`), `everyNRuns?` (1–100, default 5), `maxHops?` (1–64, default 30), `probesPerHop?` (1–5, default 3), `probeTimeoutMs?` (100–5000, default 1000) `}`. A trace also runs whenever a passing check starts failing.
- `keepBodyExcerpt?` — keep the first 4 KB of the response on every run, not only on failures.
- `scope?` — the automation device-filter tree (`{ "allAssets": true }` for every agent host), and `assetIds?` (≤2000) to pin hosts regardless of the filter. At least one of the two must select something.

`409` when the new check would be the 51st enabled one. A host runs at most 20 checks, the oldest first; past that, the newer ones stay listed as members but are not run, and a `path_check.agent_over_cap` event is written.

```bash
curl -X POST -H "Authorization: Bearer $POLARIS_TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"ERP web","kind":"https","target":"https://erp.example.com/health","http":{"expectStatus":"200"},"scope":{"allAssets":true}}' \
  "$POLARIS_URL/api/v1/path-checks"
```

### PUT /path-checks/:id

_Gate: pathChecks:write_

Replace a check — the full body `POST` takes, not a patch. Agents pick the new definition up on their next heartbeat.

### POST /path-checks/:id/enabled

_Gate: pathChecks:write_

`{ enabled: boolean }`. A disabled check keeps its hosts and their last results; agents stop running it.

### DELETE /path-checks/:id

_Gate: pathChecks:write_

`204`. Removes the check and its host list.

### POST /path-checks/preview-sources

_Gate: pathChecks:write_

Dry-run a Sources selection without saving: body `{ scope?, assetIds? }`. Returns `{ total, pinned, matchedWithoutAgent, agents[], pinnedWithoutAgent[], minAgentVersion }`; `agents` (first 100) is `{ assetId, hostname, ipAddress, os, agentVersion, online, supported, pinned, pinnedOnly }`.

### GET /path-checks/filter-schema

_Gate: pathChecks:read_

The fields, operators and option lists the `scope` filter accepts — the same vocabulary as an automation's device filter.

### GET /assets/:id/path-checks

_Gate: assets:read_

`{ checks: [...] }` — the checks this host runs, each with its definition, `latest` (the result fields listed under `/results`) and `latestSample` (the newest run in full, including `bodyMatched, bodySha256, bodyBytes, bodyExcerpt, tlsNotAfter, tlsIssuer`). Empty when the host runs none.

### GET /assets/:id/path-check-history?checkId=

_Gate: assets:read_

One check's series from this host. `checkId` is required; `range=` `1h | 12h | 24h | 7d | 30d` (default 24h), or `from=` + `to=` (ISO, ≤1 year). Returns `{ range, checkId, since, until, tier, bucketSeconds, samples }`. On the `detail` tier a sample is one run: `{ timestamp, ok, latencyMs, dnsMs, connectMs, tlsMs, ttfbMs, httpStatus, hopCount, error }`, with a timing field `null` when it was not measured. On `hourly` / `daily` each sample is a bucket carrying averages plus `okCount, failCount, sampleCount, minLatencyMs, maxLatencyMs`.

### GET /assets/:id/path-check-traceroutes?checkId=

_Gate: assets:read_

The newest traceroutes this host ran for one check, newest first: `limit=` 1–50, default 10. Returns `{ checkId, traceroutes }`; each is `{ timestamp, destinationIp, complete, hopCount, pathHash, reason` (`scheduled` or `transition` — taken because the check started failing), `note, hops }`. A hop is `{ ttl, ip, rdns, rttMs[] }` (`ip: null` and `-1` RTTs for probes that got no reply), plus `assetId, hostname, monitorStatus, interfaceName, subnetCidr` when the address belongs to something Polaris knows, as it stood when the trace was stored.
