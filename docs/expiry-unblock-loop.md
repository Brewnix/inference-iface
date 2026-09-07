# Expiry → unblock → receipt loop

**Status:** working spec (2026-09-07)  
**Schemas:** pin this repo (`fyber.inference_iface/v0`, `fyber.receipt/v0`, `fyber.common/v0`) — **no v0 schema break**  
**Companion:** [Zero-LLM OPNsense detect → block → receipt loop](zero-llm-opnsense-loop.md)

Goal: complete the contain loop so `ai_autoblock` members do not rot after `ttl_s`. Site-local. The plane is **not** required. Same envelopes and receipts as the detect → block path; a later local model can fill the same `firewall.unblock_ip` shape without changing the executor.

## Components (site box)

| Piece | Role |
|-------|------|
| **TTL ledger** | Beside the receipt store. On each **successful** `firewall.block_ip`, record `(ip, alias, expire_at, parent_receipt_id, call_id)`. SQLite or JSONL. |
| **Expiry actor** | `kind: "rule"`, `id: "brewnix-rules/expiry"` — **not** Suricata-driven. Ticks the ledger; emits an envelope when `expire_at` has passed. |
| **Policy** | Allowlist + expiry exceptions below → `observe` \| `execute` (no rate-limit hold on this path). |
| **Executor** | OPNsense `firewall/alias_util` **remove** member. |
| **Receipt store** | Same append-only `fyber.receipt/v0` JSONL + hash chain as the companion loop. |

Ledger lives next to receipts, e.g. `/var/lib/brewnix/ttl_ledger.jsonl` (or a SQLite file in that directory). Do not store EVE payloads.

Expiry is a **separate** rule pack from `brewnix-rules/v0.1` (IDS) and from [health-watch v0.1](health-watch-v0.md) (non-IDS). It does not read `feature_bundle` alert counts.

### Envelope (`fyber.inference_iface/v0`)

```json
"actor": { "kind": "rule", "id": "brewnix-rules/expiry", "purpose": "triage" }
```

| Field | Value |
|-------|--------|
| `judgment.severity` | `info` |
| `judgment.classes` | echo the parent contain cycle, or `["unknown"]` |
| `judgment.subjects` | the expired IP (`kind: "ip"`) |
| `confidence` | `1.0` (rules) |
| `needs_human` | `false` |

`inputs_digest` still binds a canonical digest. Build a **ledger snapshot** bundle (empty alert counts; `prior_blocks` for the expired member with `remaining_ttl_s: 0`) — not an EVE window. Envelope must include the same required fields as the companion loop.

#### Proposal (one per expired member)

```json
{
  "call_id": "<uuid>",
  "tool": "firewall.unblock_ip",
  "mode": "execute",
  "reason_code": "ttl_expired",
  "args": {
    "ip": "203.0.113.50",
    "alias": "ai_autoblock"
  }
}
```

Shape reference: `examples/expiry.unblock.example.json`.

### Executor

For each proposal with policy decision `execute`:

1. Call OPNsense `firewall/alias_util/delete/ai_autoblock` (or `remove`) with `{ "address": "<ip>" }`. Prefer the same runtime-table path as add; do not rewrite WAN rules.
2. Append `execution[]`: `status` **`expired`** (preferred) or `applied`, `executor: "opnsense-api@site"`, `effect: { "alias", "ip", "cause": "ttl_expired" }`.
3. Advance the ledger row (cleared / consumed) so a later tick cannot re-emit.

**Dedupe:** if the IP is already gone from the alias (manual remove, prior expiry, never added) → policy `observe` (+ optional `receipt.annotate`). Still **advance the ledger**. Do not fail the cycle.

### Receipt (`fyber.receipt/v0`)

| Field | Value |
|-------|--------|
| `door` | `site_defense` |
| `purpose` | `contain` |
| `policy.decision` | `execute` on a real remove; `observe` on dedupe |
| `policy.engine` | `brewnix-policy/v0` |
| `policy.rule_ids` | `["ttl_expired"]` |
| `execution.status` | preferred `expired` (or `applied`) on remove |
| `human` | `required: false`, others `null` |
| `parent_id` | ledger `parent_receipt_id` (the block receipt) |
| `integrity.prev_hash` | prior receipt `body_hash` (site chain continues; may not equal parent if other receipts landed in between) |
| `integrity.sig` | `null` (v0) |

Same hash algorithm as the companion loop. Persist on the same JSONL. Never store packet captures or prompts.

## Policy (`brewnix-policy/v0`)

1. **Never auto-unblock from IDS noise.** Only (a) this ledger expiry actor or (b) an **explicit** `firewall.unblock_ip` (e.g. a later auditor-approved early unblock). `brewnix-rules/v0.1` must not emit unblock from alert counts.
2. **Rate limits do not apply** to expiry unblocks (or use a separate, much higher cap). Otherwise a hold traps expired members in the alias forever.
3. **Whitelist after block:** expiry still attempts a no-op remove / `observe`. Do not leave the ledger stuck because the IP is now whitelisted.
4. Reject unknown tools / bad args (JSON Schema). Extra keys on `ArgsFirewallUnblockIp` fail validation.

Auditor / notify is **not** on the expiry hot path. Early human unblock is a different cycle (`firewall.unblock_ip` from the notify door); a later expiry tick then dedupes.

## Locked tool args (`firewall.unblock_ip`)

From `schemas/common.v0.json` → `$defs/ArgsFirewallUnblockIp`. Do not extend.

| Field | Required | Constraint |
|-------|----------|------------|
| `args.ip` | yes | string 1–64 |
| `args.alias` | no | default `ai_autoblock`; max 64 |

`additionalProperties: false`. No `direction`, no `ttl_s` on unblock args.

## Acceptance tests

1. Block with `ttl_s: 60` → after 60s exactly **one** unblock receipt for that IP; alias has no member for it.
2. Second expiry tick for the same ledger row → `observe` (dedupe); ledger already advanced.
3. `plane_reachable: false` → expiry still runs; receipt still appended. No notify required.
4. Chain unbroken: block receipt → expiry receipt with `integrity.prev_hash` = prior `body_hash`.
5. JSON Schema validate envelope + receipt (local `$id` → `schemas/` map). Unblock `args` are only `ip` (+ optional `alias`). Extra args keys fail.

## Non-goals

- Sliding TTL refresh from new alerts — v0.1 keeps the original `expire_at`
- Auditor as the primary unblock path (expiry is automatic; auditor early-unblock is optional and explicit)
- New v0 schema fields on the ledger or on `ArgsFirewallUnblockIp`
- Suricata / IDS rules emitting `firewall.unblock_ip`

## Suggested implementation layout

```
brewnix-site-defense/
  ledger/ttl.jsonl             # or sqlite; beside receipts
  rules/expiry.py              # actor id brewnix-rules/expiry
  exec/opnsense_alias.py       # add + remove
  receipt/chain.py
  schemas/                     # pin Brewnix/inference-iface
```

## Change control

Additive clarifications to this doc are fine. Behavioral changes (auto-unblock from IDS, applying rate-limit holds to expiry, sliding `expire_at`, or putting notify on the expiry hot path) need an explicit amendment. Semantic changes to locked schemas remain **v1** or a Chris-approved schema amendment.
