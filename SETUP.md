# Setup

How to build, deploy, and run the system. Two parts: the **agent + governance
backend** (Rust), and the **public surface** (Cloudflare Worker + SvelteKit,
with the Akash validation sandbox and the governed MCP). No secrets or hosts
are hard-coded anywhere — everything reads from gitignored env files.

## Prerequisites

| Need | For |
|---|---|
| [Bun](https://bun.sh) ≥ 1.3 | the public surface, scripts, runner/bridge |
| [wrangler](https://developers.cloudflare.com/workers/wrangler/) (installed via `bunx`) | deploy the Worker |
| A Cloudflare account on a **paid Workers plan** | Durable Objects + Workers AI (the planning model is not on the free tier) |
| An Akash **Console API key** (`ac.sk.…`) | deploy the relay + validation sandbox |
| Semgrep, Rust toolchain, Python 3 | the backend agent loop (see `README.md`) |

## Environment files (all gitignored)

Create `governed-action/.env` for the backend + Akash:

```sh
AKASH_API_KEY=ac.sk.…            # Akash Console API key
AKASH_NETWORK=https://api.akashnet.io
THREATFOX_AUTH_KEY=…             # abuse.ch ThreatFox
CLOUDFLARE_PAGES_PROJECT=…       # the single allowed publish target
```

The public surface uses two more, created for you by a script (below):
`sentinel-board/.dev.vars` and `sentinel-board/.runner.env`. All three are in
`.gitignore`; never commit them and never echo their values.

---

## Part A — Agent + governance backend

Builds the agent loop, the governance engine, the signed ledger, and the
decision relay. Full detail in `README.md`; the short path:

```sh
export CARGO_TARGET_DIR=/path/to/governed-forge/.target   # keep artifacts off the OS disk
python3 scripts/fetch-osv-dataset.py     # OSV dataset + NVD 2.0 corroboration index
cargo build --offline --locked
./run-loop.sh                            # feeds → classify → gate → ledger → relay
```

The relay (and the validation sandbox, Part C) run on Akash; the backend
handles that deployment. The public surface only needs the relay reachable to
the bridge (Part B).

---

## Part B — Public surface (sentinel-board)

A single Cloudflare Worker serves the static site, the live hub (Durable
Object), the `/api/*` endpoints, and the governed MCP at `/mcp`.

```sh
cd sentinel-board
bun install
```

### 1. Generate the ingest token (once)

```sh
bun scripts/init-secrets.ts              # writes INGEST_TOKEN into .dev.vars + .runner.env
```

This token authenticates the bridge → hub push. It is written to gitignored
files only and never printed.

### 2. Build and deploy the Worker

```sh
bun run build
CLOUDFLARE_ACCOUNT_ID=<your-account-id> bunx wrangler deploy
```

The deploy reports the live URL (`https://<name>.<subdomain>.workers.dev`) and
the bindings (Durable Object, Workers AI, Assets).

### 3. Set the production secret

```sh
bun scripts/put-secret.ts INGEST_TOKEN   # pipes the token from .dev.vars, never echoes it
```

### 4. Run the bridge (mirror the real relay into the hub)

The bridge reads the real decision relay and forwards it to the hub so the site
shows the **actual** pipeline — not a separate run. Point it at the relay and
the Worker URL in `.runner.env` (`RELAY_HTTP`, `LIVE_URL`), then run it as a
supervised service:

```sh
systemd-run --user --unit=hackerbot-bridge -p Restart=always \
  --working-directory="$PWD" bun runner/bridge.ts
```

Counters on the site reconcile exactly to the relay totals.

### 5. Custom domain (optional)

Add the domain through the Worker's **Custom Domains** (dashboard, or
`routes: [{ pattern: "<host>", custom_domain: true }]` in `wrangler.jsonc` with
`workers_dev: true` kept). Cloudflare provisions the DNS record and
certificate; **do not** add a manual CNAME to `workers.dev` (it conflicts and
won't route). Cert activation can take up to ~an hour; the `workers.dev` URL
serves in the meantime.

---

## Part C — Akash validation sandbox

Validates single-source advisories by reproduction, off-host. Deploys through
the Akash Console API using `AKASH_API_KEY` from `.env`.

```sh
cd sentinel-board
bun scripts/akash-deploy.ts              # create → accept bid → wait → health
```

The script writes `HERMES_SANDBOX_URL` + `HERMES_SANDBOX_TOKEN` into
`.runner.env`. Test it:

```sh
bun scripts/test-sandbox.ts              # health + two real validations
```

Run the continuous validation loop (picks real single-source advisories,
validates each on the sandbox, streams to the board):

```sh
systemd-run --user --unit=hermes-loop -p Restart=always \
  --working-directory="$PWD" bun runner/hermes-loop.ts
```

Close the sandbox when done (stops Akash billing):

```sh
bun scripts/akash-deploy.ts --close <dseq>
```

---

## Part D — Governed MCP

Served at `/mcp` as soon as the Worker is deployed — no extra steps. Any MCP
agent connects over Streamable HTTP:

```sh
claude mcp add --transport http hackerbot https://<your-host>/mcp
```

Test it directly:

```sh
bun scripts/test-mcp.ts https://<your-host>/mcp
```

To route decisions (and signed receipts) through the real governance gateway,
set its URL as a Worker secret — no redeploy needed for the UI:

```sh
echo <gateway-url> | bunx wrangler secret put GATE_URL
```

Until then the MCP evaluates the same affected-range rule the policy encodes
and labels decisions as rule-evaluated (never a synthetic signed receipt).

---

## Verify

```sh
curl -s https://<host>/api/state | jq '.counters'     # live counters
curl -s https://<host>/api/board | jq '.advisories | length'
bun scripts/test-mcp.ts https://<host>/mcp            # governed tool calls
```

Running services (auto-restart, survive logout):

```sh
systemctl --user status hackerbot-bridge hermes-loop
```

## Teardown

```sh
systemctl --user stop hackerbot-bridge hermes-loop
bun scripts/akash-deploy.ts --close <dseq>            # stop the sandbox lease
bunx wrangler delete                                   # remove the Worker (optional)
```
