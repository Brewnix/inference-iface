# Zero-LLM OPNsense detect → block → receipt loop

**Status:** working spec (2026-09-06)  
**Schemas:** pin this repo (`fyber.inference_iface/v0`, `fyber.receipt/v0`, `fyber.feature_bundle/v0`, `fyber.common/v0`)  
**Actor:** `kind: "rule"` only — no LLM, no SaaS on the hot path

Goal: Suricata / OPNsense alerts → contain → receipt, using the same envelopes a later local model can fill without changing the executor.

## Components (site box)

| Piece | Role |
|-------|------|
| **Sensor** | Suricata EVE (or OPNsense IDS alerts) → redacted `fyber.feature_bundle/v0` |
| **Rules engine** | Emits `fyber.inference_iface/v0` with typed `proposals` |
| **Policy** | Allowlist, severity map, rate limits → `observe` \| `propose` \| `execute` \| `hold_human` |
| **Executor** | OPNsense API only (`firewall.block_ip` / `firewall.unblock_ip`) |
| **Receipt store** | Append-only JSONL of `fyber.receipt/v0` (+ hash chain) |

## Preconditions (one-time OPNsense)

1. Alias `ai_autoblock` (type: Host(s)).
2. WAN rule: block source in `ai_autoblock` (created once; never rewritten per event).
3. API user with rights to `firewall/alias_util` (add/remove) — not full admin if avoidable.
4. Optional cron / job queue: expire members when `ttl_s` elapsed (executor records `status: "expired"`). Full loop: [Expiry → unblock → receipt](expiry-unblock-loop.md).

## Pipeline

```
EVE/alerts ──► feature_bundle ──► rules → envelope ──► policy ──► executor ──► receipt
                 (digest)            actor=rule              APIs only
```

### 1. Feature bundle (every cycle, e.g. 60–300s)

Build `fyber.feature_bundle/v0` from the window:

- `counts.alert_total`, `by_sid`, `by_src`
- `top_subjects[]` (`ip`, `hits`, `ports`, `sids`)
- `health` (`cpu` / `disk` / `wan_gateways` / `suricata`) — consumed by the sibling **non-IDS** pack [health-watch v0.1](health-watch-v0.md), not by this contain loop
- `whitelist_hits`, `prior_blocks` (current alias members + remaining TTL)

`features_digest` = `sha256:` + SHA-256 of **canonical JSON** of that bundle.  
Envelope `inputs_digest` = the same value.

See `examples/feature_bundle.example.json`.

### 2. Rules → envelope (`fyber.inference_iface/v0`)

```json
"actor": { "kind": "rule", "id": "brewnix-rules/v0.1", "purpose": "triage" }
```

#### Rule pack v0.1 (minimal)

| Rule id | Match | Judgment | Proposal |
|---------|-------|----------|----------|
| `port_scan_burst` | one `src` ≥ **N** distinct ports **or** ≥ **M** alerts in window on scan SIDs | `severity: critical`, `classes: ["port_scan"]` | `firewall.block_ip`, `mode: execute`, `ttl_s: 86400` |
| `ssh_brute` | auth-fail SID burst from one `src` ≥ **K** | `high` / `brute_force` | same block shape, `ttl_s: 86400` |
| `noise_ignore` | known benign SID set / whitelist IP | `info` | `proposals: []` |

**Thresholds (defaults until tuned):** N = 10 distinct ports, M = 20 alerts, K = 15 auth-fail events in the window. Tune with acceptance tests; do not invent SID lists in this doc — keep them in `rules/v0.1.yaml` beside the engine.

Envelope must include: `schema`, `trace_id` (new UUID per cycle), `site_id`, `observed_at`, `inputs_digest`, `judgment`, `proposals`, `confidence` (rules: `1.0` or fixed `0.9`), `needs_human` (`false` for auto rules).

#### Hard constraints

- Every `judgment.subjects[].value` and block `args.ip` must appear in `feature_bundle.top_subjects` or `counts.by_src`.
- Auto path tools in this loop: `firewall.block_ip` and `firewall.unblock_ip` only.
- No `shell.exec`, no new pf rules per event, no Suricata SID toggles in v0.1.
- Validate every envelope against `schemas/inference_iface.v0.json` (+ `common.v0.json`).

Shape reference: `examples/envelope.example.json` (swap `actor.kind` to `rule` and `actor.id` to `brewnix-rules/v0.1`).

### 3. Policy (`brewnix-policy/v0`)

On each envelope:

1. Reject unknown tools / bad args (JSON Schema).
2. If IP in static whitelist → `observe`.
3. If IP already in `ai_autoblock` with remaining TTL → `observe` (dedupe).
4. Rate limit: max **B** new blocks / hour / site (default B = 30) → else `hold_human` (**requires** `notify.operator`).
5. Else if `severity` is `critical` and rule id ∈ auto set (`port_scan_burst`, …) → `execute`.
6. Else if `severity` is `high` → **`propose` only** in v0.1 (**requires** `notify.operator`; no auto block). Critical-only auto keeps false positives recoverable.

Notify door (channel `fyber.auditor`, queue-if-offline, human resolution): [notify.operator ↔ shared auditor door](notify-operator-audit-door.md).

Record `policy.rule_ids` as the matching detector ids (e.g. `["port_scan_burst"]`).  
`policy.engine` = `brewnix-policy/v0`.

### 4. Executor

For each proposal with policy decision `execute`:

1. Call OPNsense `firewall/alias_util/add/ai_autoblock` with `{ "address": "<ip>" }` (runtime table; prefer paths that do not require a full ruleset reload).
2. Enqueue removal at `now + ttl_s`.
3. Append `execution[]`: `status` `applied` \| `failed`, `executor: "opnsense-api@site"`, `effect: { "alias", "ip", "ttl_s" }`.

**Expiry:** a rules actor `id: "brewnix-rules/expiry"` emits `firewall.unblock_ip`; executor removes from alias; write a receipt with `execution.status: "expired"` or `applied` on unblock. Site-local TTL ledger + policy (no IDS-driven unblock, rate limits do not trap expired members): [Expiry → unblock → receipt](expiry-unblock-loop.md).

### 5. Receipt (`fyber.receipt/v0`)

One receipt per decision cycle (including `observe`):

| Field | Value |
|-------|--------|
| `door` | `site_defense` |
| `purpose` | `contain` if any block applied; else `triage` |
| `posture` | live: `wan_up`, `plane_reachable`, `path_b: "n/a"`, `sell_state: "off"` or `"n/a"` |
| `input.sources` | `["suricata", "opnsense"]` |
| `actor` / `judgment` / `proposals` | copy from envelope |
| `human` | `required: false`, others `null` for pure auto |
| `integrity.prev_hash` | prior receipt `body_hash` or `null` |
| `integrity.body_hash` | SHA-256 of canonical receipt **with `integrity` omitted** |
| `integrity.sig` | `null` (v0) |

Persist e.g. `/var/lib/brewnix/receipts.jsonl`. Never store EVE payloads, packet captures, or prompts.

Shape reference: `examples/receipt.example.json`.

## Worked critical path

1. Bundle shows `203.0.113.50` with 30 hits on ports 22/23 → digest bound into envelope.
2. Rule `port_scan_burst` → envelope (`actor.kind: rule`).
3. Policy → `execute`.
4. Alias add succeeds.
5. Receipt with `policy.decision: execute`, hash-chained.

## Acceptance tests

1. JSON Schema validate every envelope + receipt (local `$id` → `schemas/` map; no network fetch).
2. Synthetic EVE burst → exactly one `applied` block for that IP; second cycle → `observe` (dedupe).
3. Whitelisted IP never blocked.
4. Run with no model process present → loop still contains.
5. `plane_reachable: false` → still blocks and writes receipts.
6. After `ttl_s`, IP removed; receipt chain continues unbroken (see [expiry-unblock-loop](expiry-unblock-loop.md)).

## Non-goals

- LLM triage (later: same executor, `actor.kind: "model"`)
- `net.quarantine_host`, Hypermesh `lease_stop`, IR door
- SaaS on the detect → block path
- Duplicating schemas into other repos — **consume** this repo as source of truth

## Suggested implementation layout

```
brewnix-site-defense/          # or module under proxmox-firewall
  rules/v0.1.yaml
  rules/expiry.py              # sibling contain TTL — expiry-unblock-loop.md
  rules/health-v0.1.yaml       # sibling non-IDS pack — health-watch-v0.md
  policy.py
  sensors/eve_to_bundle.py
  exec/opnsense_alias.py
  receipt/chain.py
  schemas/                     # git submodule or release pin → Brewnix/inference-iface
```

## Change control

Additive clarifications to this doc are fine. Behavioral changes to auto-severity, tools on the hot path, or receipt integrity rules need an explicit amendment (and may require `inference_iface` / `receipt` v1 if they break the locked schemas).
