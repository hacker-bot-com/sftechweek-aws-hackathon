# Hacker Bot — a security agent that can't overstep

### ▶ Live and public — judges, start here: **https://sf.hacker-bot.com**

A defensive security agent doing real work on the open web, every action
governed. Open the site and watch it live:

- **real gate decisions streaming** — every EXECUTE and REFUSED as it happens
- **verified advisories with their fixes** — severity, fixed version, and the
  exact `cargo update` / `npm install` command to run
- **fix-diff checks on Akash** — when an advisory has only one source,
  Hacker Bot spins up a sandbox container on Akash that fetches the release
  before the fix and the fixed release and diffs the source, so you can see
  whether a fix actually shipped (no package code executed). This is context
  for the reader. It is not a reproduction of the fault, and it does not count
  as a second source.
- **"Does it hit your stack?"** — paste your own `Cargo.lock` /
  `package-lock.json` / `requirements.txt` and get the advisories that affect
  your exact versions, with the upgrade command for each

The public site is a static Cloudflare Worker + Durable Object hub; it mirrors
the real decision stream and never exposes ledger internals, contract source,
or infrastructure (disclosure tier 3).

---

**Every agent action passes a µLang gate: EXECUTE or REFUSED, both with a
signed receipt, every receipt on a post-quantum ledger.** The agent never
touches the target directly; the gate decides, and the receipt proves it.

This is not a mockup. It is running now, end to end:

- **2,899 real advisories** (OSV's entire crates.io dataset) gated one by one
- **A second source counts only when NVD actually holds the record.** Each
  draft's CVE is checked against a 257,731-ID index built from NVD's own 2.0
  feeds. OSV and NVD are two databases holding the same advisory; aliases of
  one advisory never count twice.
- **Semgrep scans every draft before the gate reads it** — a finding flips
  the verdict; the receipt says which rule fired
- **ML-DSA-65 signed lattice**: every EXECUTE and every REFUSED commits as a
  ledger entry; tamper one byte and verification fails (verified by an
  out-of-band-pinned checker, ledger crate only)
- **Live relay on Akash** (decentralized compute), token-protected push,
  origin-checked WebSocket — the decision stream you can watch in real time
- **ClickHouse flux**: every decision queryable by verdict, refusal class,
  sources cited, and findings count
- **Constant-time refusal**: the refusal envelope leaks nothing about WHICH
  rule fired — measured, KS test, n=2000/class (no detectable difference
  across rule classes)

## The one-line idea per sponsor

- **ClickHouse** — every EXECUTE and REFUSED is a row: query the agent by
  verdict, policy, tool, or refusal class, like any other observability data.
- **Semgrep** — a Semgrep finding is a gate INPUT, not a report you read
  later: the scan runs pre-write, and the finding count is a fact the gate
  refuses on. We shipped a custom rule that caught a real XSS in our own
  board renderer (before/after artifacts in `semgrep/`).
- **Akash** — the loop runs on decentralized compute, and the point travels:
  an agent on infrastructure nobody owns is only safe if what it did is
  verifiable. Two workloads run on Akash: the decision **relay**, and a
  **sandbox** that runs fix-diff checks on single-source advisories — it
  fetches the release before the fix and the fixed release and compares the
  source, without executing any package code.
- **Docker Hub** — the relay image ships from a private repo; the provider
  pulls it with credentials sealed end to end (JWE, never in the SDL).
- **Cloudflare** — the public site at **sf.hacker-bot.com** is a Worker +
  Durable Object that mirrors the live decision stream and fix-diff checks.
  The runner posts each cycle's board to the site's token-protected ingest.
  The agent's own deploy path is gated: a deploy to any project other than
  the board is refused before wrangler runs.

## What the gate refuses (all live, all receipted)

| Attack | Why it's refused |
|---|---|
| Uncorroborated advisory (1 source) | publishing needs two databases holding the record (OSV + NVD); an alias of the same advisory counts once |
| Out-of-scope deploy (real wrangler target, fully corroborated) | the scope clause alone stops it — `wrangler NEVER ran` is the logged outcome |
| Dangerous content (curl\|sh, rm -rf /, pickle.loads, private key) | 4 Semgrep errors → REFUSED, `refusal_class=semgrep_findings` |

And the refusals are **constant-time**: an attacker cannot probe which rule
blocked them, or how close they got.

## Verify it yourself

```sh
# the full loop: feeds → semgrep → gate → flux → relay → board
./run-loop.sh

# verify the ledger against the out-of-band pin (ledger crate only,
# never saw the build):
GA_COMMIT=<hash> ga-verify demo-artifacts/lattice.json "$(cat demo-artifacts/pin.txt)"

# query the flux (independently, over HTTPS):
SELECT verdict, refusal_class, sources_cited, semgrep_findings
FROM gate_flux.gate_flux ORDER BY ts DESC
```

## Repo layout

```
crates/        ga-schema, ga-gate, ga-ledger-bridge, ga-sentinel (loop),
               ga-flux (ClickHouse), ga-feeds (OSV+ThreatFox), ga-relay,
               ga-verify (L10 independent verifier), ga-bench (K2–K7 harnesses)
contracts/     GA-SENTINEL.ul (the publish contract), GA-DISCLOSURE.ul
               (circle of trust), GA-META-SECURITY, GA-DOS, GA-JACOBIAN
semgrep/       sentinel-rules.yml — the live classifier + the XSS before/after
prereg/        PREREG.md — kill rows K1–K10 locked BEFORE measurement (G3 blind pass green)
scripts/       fetch-osv-dataset.py — OSV dataset + NVD index builder
akash/         deploy_relay.py — seal → create → bid → lease → live (one command)
sentinel-board/  the live site: worker/ (Cloudflare Worker + Durable Object hub),
               runner/ (continuous feed monitor + back-to-back gated cycles),
               sandbox/ (the Akash fix-diff checker), src/ (the site itself)
```

## Built at the event vs. prior work

- **Built during the hackathon (after the 11:00 kickoff):** the Hacker Bot
  agent loop (`ga-sentinel`), the feeds (OSV, the NVD index, ThreatFox), the
  Semgrep classifier, the GA-SENTINEL publish contract, the ClickHouse flux,
  the relay, the live site and runner, and the Akash fix-diff sandbox.
- **Prior work we built on:** the Governed Action core (gate, action schema,
  ledger bridge, independent verifier, the META-SECURITY / DOS / JACOBIAN /
  DISCLOSURE contracts and the K-row harnesses), started before kickoff and
  hardened during the day, plus our µLang compiler and `ulang-ledger`.

## Honest fences (stated before anyone asks)

- The signing key is on-box today; the pin detects POLICY swaps, not a
  root-key receipt forgery. Off-host key is the named next step.
- Timing rows were measured on an unloaded machine; a contended run reads
  RED — that's in PREREG as an environment rule, not a bug.
- The syscall layer (L1a eBPF) is stubbed; the demo shows the agent/API
  layer, and the one-liner says agent tool calls, not syscalls.
- A fix-diff check shows that a fix release changed the source. It does not
  reproduce the fault and is not corroboration. Today the runner also shows
  single-source advisories that pass it on the board, marked separately and
  with no gate receipt; moving that check into the gate as a fact is the next
  change.
- OSV and NVD records often trace back to the same original advisory (for
  example, a GitHub-assigned CVE). We claim two databases hold the record,
  not two independent investigations.

## Build

```sh
export CARGO_TARGET_DIR=/media/jake-strongclose/B84E19B14E196A002/ulang-mvp/governed-forge/.target
python3 scripts/fetch-osv-dataset.py   # OSV dataset + NVD index
cargo build --offline --locked
```
