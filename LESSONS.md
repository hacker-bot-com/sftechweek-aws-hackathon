# Lessons

What we learned building a governed autonomous agent and its public surface in
a day. Candid, so the next build is faster.

## Architecture

**Keep the model out of the enforcement path — and prove it, don't claim it.**
Every PASS/FAIL comes from the deterministic governance engine; models only
draft, classify, and summarize. This isn't a slogan — it's what let us put a
reasoning model in the loop without ever trusting it to decide. When the
planning model returned garbage, nothing unsafe happened: its output is a fact
the gate weighs, not a verdict.

**Mirror the source of truth; never run a parallel pipeline.** The first public
site spawned its own agent runs and looked "fresh" — disconnected from the real
2,800-advisory pipeline. The fix was a thin, trusted *bridge* that forwards the
real decision relay into the public hub. The public surface should *reflect*
the system, not re-run it. Everything shown now reconciles exactly to the relay.

**Counters must reconcile to the source.** We briefly showed inflated publish
counts because the bridge emitted both a "publish decision" *and* a
"published advisory" event per advisory. Pick one source of truth per metric
and derive the rest. If a number on screen doesn't match the upstream total,
it's wrong — even if it looks impressive.

**Fail closed.** A dead pre-write scanner is treated as "findings present," not
"clean." A missing corroboration index refuses, it doesn't wave things through.
The safe default is the restrictive one.

## Validation by reproduction

**Diff the fix against the version *before* it, not the first-affected one.**
Our first validator compared `openssl 0.10.0 → 0.10.79` and proudly reported
"90 files changed" — which is 79 releases of unrelated churn, not the fix. The
honest signal is the fixed version vs. its immediate predecessor: that isolates
the actual fix changeset (`0.10.78 → 0.10.79`, a handful of files). Precision
is the difference between evidence and noise.

**Name what you actually proved.** Fix-diff confirms *a real fix shipped at the
stated version boundary*. That is strong first-hand corroboration — but it is
not exploit reproduction, and we label it as exactly what it is. Overclaiming
is the fastest way to lose a security audience.

**Do the dangerous work somewhere disposable.** Building arbitrary package
versions is exactly the kind of thing that can fill a disk or run untrusted
code. It runs on ephemeral, isolated Akash containers — never the production
host — and the sandbox only fetches and diffs source, it never builds or
executes package code.

## Operations & security

**Never surface an internal host.** We leaked a sandbox provider's host:port
onto the public page. The lesson is defense in depth: stop sending it at the
source, *and* strip it server-side so a bad input can't reintroduce it, *and*
purge the cached state that already had it. One scrub layer is not enough when
it's already public.

**Authenticate every write and check origins on every stream.** The push
endpoint is bearer-token gated; the live socket is origin-checked. A public hub
that anyone can write to is not a mirror of the gate — it's a defacement vector.

**Custom domains on Workers are not CNAMEs.** Manually CNAME-ing a hostname to
`workers.dev` doesn't route and conflicts with Cloudflare's own custom-domain
provisioning. And adding a `routes` block *without* `workers_dev: true` silently
disabled the `workers.dev` URL and took the site down. Use a real Custom Domain,
keep `workers_dev: true`, and expect certificate activation to take time.

**Secrets never touch the command line or the logs.** Tokens are generated into
gitignored files and piped to `wrangler secret put` over stdin — never echoed,
never an argument, never committed.

## Models

**Don't put a flaky model on a critical path.** A reasoning model wouldn't
reliably return structured JSON even with a schema. Because the deterministic
sandbox stands on its own, the model stays advisory — a planning hint, not a
dependency. Design so the unreliable component can fail without failing the
system.

## The governed MCP

**Let judges govern their own agent.** The strongest demo of "every action is
governed" is handing someone the controls: connect any MCP agent, ask it to add
a dependency, and watch the gate refuse with the CVE and the fix. A governed
MCP (Streamable HTTP, stateless) turns a claim into something they can try, and
every call lands on the live board.

**Build the seam before you need it.** The MCP works today on a transparent
rule and labels itself "gate receipt pending." When the real governance gateway
is exposed, one secret (`GATE_URL`) swaps in signed receipts with no redeploy.
A clean seam between lanes beats a blocked lane.

## Process

**Read the actual rules, not a summary.** Early on we optimized against a
paraphrase of the brief and built the wrong thing — twice. Pull the real source,
read it literally, then build. A day is too short to build the wrong thing fast.

**Ship the product, not a dashboard of it.** The ask was a working agent doing
real work on the open web — not a pretty view of one. When in doubt, make the
thing judges can *use* (check their stack, connect their agent), not just watch.

**Honest beats polished.** No synthetic receipts, no invented data, hosts
scrubbed, claims matched to evidence. For a security project, the credibility
of what you *didn't* fake is the whole game.
