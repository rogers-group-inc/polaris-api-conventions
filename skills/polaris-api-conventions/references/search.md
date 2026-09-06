# Search

_Generated from Polaris `public/api.html` (section `search`). Do not edit by hand — re-run `scripts/import-api-html.mjs`._

### GET /search?q=

_Gate: per-group read keys · filter-don't-403_

Global typeahead across six groups. Response: `{ query, blocks, subnets, reservations, assets, ips, sites }` — every group always present; groups the role can't read (or that didn't match) are empty arrays. Group gates: blocks→`ipBlocks`, subnets→`subnets`, reservations→`reservations`, assets→`assets`, sites→`deviceMap`; the `ips` cross-reference needs both `subnets` and `reservations` read.

- Minimum 2 characters (1 when scoped). Terms are AND-combined; `"quoted phrases"` stay whole; single terms get IP/CIDR/MAC special-casing.
- Scope prefixes: `block:`/`b:`, `network:`/`n:`, `asset:`/`a:`, `reservation:`/`r:`, `map:`/`m:`, `tag:`/`t:` (tag spans blocks + networks + assets).
- Caps: 8 hits per group unscoped; 200 for a scoped query. No paging — scope the query instead.
