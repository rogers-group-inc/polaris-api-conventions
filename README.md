# polaris-api-conventions

A Claude Code plugin for developers building applications that **consume the Polaris REST
API** (`/api/v1`): SIEM-driven quarantine, NOC wallboards, inventory consumers, IPAM
automation. It is the plugin form of the developer page Polaris serves at `<polaris-url>/api`.

## Use it in another project

```
claude --plugin-dir C:\Users\dmoore\VSCode\polaris-api-conventions
```

The skill `polaris-api-conventions` then auto-loads whenever a task calls a Polaris server,
needs a token, quarantines a device, builds a wallboard, or asks what an endpoint returns.
Invoke it by hand with `/polaris-api-conventions:polaris-api-conventions`.

## Regenerate after Polaris changes its API page

`skills/polaris-api-conventions/references/*.md` are **generated** from the Polaris checkout's
`public/api.html` — the curated external contract. Never edit them by hand:

```
node scripts/import-api-html.mjs <path-to-polaris>/public/api.html
```

The script rewrites the references, one file per section, and stamps the source file's
sha256 into `.claude-plugin/plugin.json` (`sourceApiHtmlSha256`). Bump `version` when you
publish a regenerated set. `SKILL.md`, `examples/` and this README are hand-maintained.

## Layout

```
.claude-plugin/plugin.json
skills/polaris-api-conventions/SKILL.md          # the contract in short + routing
skills/polaris-api-conventions/references/*.md   # generated, one per api.html section
skills/polaris-api-conventions/examples/         # minimal clients (TypeScript, PowerShell)
scripts/import-api-html.mjs
```
