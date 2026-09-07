# fyber.privilege_grant v0

**Status:** working spec — **LOCKED 2026-09-07** (Chris)  
**Schemas:** pin this repo (`fyber.inference_iface/v0`, `fyber.receipt/v0`, `fyber.feature_bundle/v0`, `fyber.common/v0`) — **no v0 schema break**  
**Plane contract:** `fyber.privilege_grant/v0` (docs-first; **not** a file under `schemas/`)  
**Companion:** [fyber.auditor API v0](fyber-auditor-api-v0.md) (one-shot held-tool ticket) · [notify.operator ↔ shared auditor door](notify-operator-audit-door.md) (site half)

Goal: a **bounded, expiring policy elevation** for one `(site_id, incident_id)`. Automations **propose**. A human (or dual-control) **mints**. The site hot-reloads allowlists / budgets / `rails_profile` for that incident only. Executions stay on `fyber.inference_iface/v0` + the typed executor. The LLM never executes.

A grant is **not** a one-shot held-tool approve. That stays on `fyber.auditor.ticket/v0`. A ticket **may** approve a grant (especially `break_glass`). Empty `asks` → refuse the grant; use a ticket.

**Home:** this repo holds the contract (MIT / OSS intent). **Panopticon** implements the grant store / gateway. Not hyperme.sh the marketing site. Site JSON Schemas stay locked; do not add grant fields to `schemas/`.

## Goal / non-goals

| Goal | Meaning |
|------|---------|
| Bounded elevation | Change policy allowlists, token budget, and/or `rails_profile` for one `(site_id, incident_id)` with a **mandatory** expiry. |
| Same ask kinds | `break_glass` is the shortest TTL + loudest audit + wider **max catalog of the same five ask kinds** — not a bypass bit, not an unbound shell. |
| Ticket vs grant | One-shot companion approve stays on the auditor ticket. A ticket may mint / ack a grant. |
| Profile composes | Named profile sets the max catalog. `asks` are optional deltas ⊆ that max. Approve may drop asks (amend); never expand past the profile. |
| Contract here | Panopticon stores and gates. This repo does not ship a schema file in v0. |

| Non-goal | Why |
|----------|-----|
| JSON Schema under `schemas/` | Separate lock. Implement against this file. |
| `incident_id` binding document | Field is required on the grant; the incident object / store is out of band. |
| SociACL Check grants | Different plane. README non-goal unchanged. |
| Enterprise compliance pack | Dual-control is a ladder rule, not a SOC2/ISO artifact in this repo. |
| Expanding `ToolName` casually | Grant tools ⊆ the locked enum and/or a named pack. New tools are a schema amendment. |
| Per-ask TTL / dual budget meters | Single grant TTL. `budget_tokens` is `max_tokens` only. |

## Axioms

1. **Automations propose; human (or dual-control) mints.** The LLM never executes and is never `resolved_by`.
2. **Grant changes policy for one incident.** Allowlists / budgets / `rails_profile` apply only to `(site_id, incident_id)`. Expiry is mandatory.
3. **Break-glass is not a bypass.** Same five ask kinds, shorter TTL, louder audit, wider max catalog. Not `shell_unrestricted`. Not “skip receipts”.
4. **One-shot held-tool approve stays on the ticket.** `asks: []` is invalid — refuse the grant and use `fyber.auditor.ticket/v0`.
5. **Profiles compose asks.** A set profile is the ceiling. Asks are optional deltas ⊆ that catalog. `rails_profile_requested` null → treat asks as deltas on **strict**.
6. **This repo is the contract.** Panopticon implements the store / gateway. MIT / OSS intent.

## Ask kinds (v0 ONLY — five)

Unknown `kind` → reject the grant. Extra keys on an ask fail validation. CUT kinds below are not accepted as aliases.

| `kind` | Fields | Meaning |
|--------|--------|---------|
| `tool_allowlist_add` | `tools` (array, min 1) | Each entry is a locked `ToolName` **or** a pack id (`packs/…`). Those tools become **execute-eligible** under the active profile for this incident. **Cannot remove** tools. Cannot imply `net.quarantine_host` or `hypermesh.lease_stop` — those need an explicit entry when the site is ready. |
| `rate_limit_raise` | `metric`, `limit` | `metric` enum is **only** `blocks_per_hour`. `limit` is a positive int (new cap). No other meters. |
| `budget_tokens` | `max_tokens` | Token cap only (**not** USD). **Requires** a `model_tier` ask on the **same** grant. |
| `model_tier` | `tier` | `local_small` \| `local_large` \| `plane_ir` \| `host_leased`. **PAIR is an engine under `local_*`, not a tier.** |
| `prompt_route` | `template_ids` (array, min 1) | Ids ⊆ the frozen template registry (aimonitoring-style route, gated by this grant). Unknown id → reject. |

`tool_allowlist_add.tools` ⊆ locked `ToolName` (`schemas/common.v0.json`) and/or pack ids. Packs are versioned names (`packs/emergency-v0`). A pack **may ship empty**; an empty pack adds nothing.

### Deferred (not v0)

`mcp_allowlist` · `knowledge_pack` (until a versioned pack registry) · `blast_radius: org` · per-ask TTL · dual budget meters (USD + tokens).

### CUT (never v0; not aliases)

`rails_profile` as an ask kind (profile is a **field**, not an ask) · `shell_unrestricted` · `disable_receipts` · `skip_human` · `exfil_ok` · `trust_plane_execute`.

## `rails_profile` ladder

| Profile | Max catalog (defaults) | Max TTL | Dual-control |
|---------|------------------------|---------|--------------|
| `strict` | Site baseline only (zero-LLM contain set + locked notify). No elevation catalog. | — (no elevated grant; use a ticket for one-shot) | — |
| `ir_elevated` | `tool_allowlist_add` ⊆ `{health.restart_service, notify.operator}` + `ids.suricata_pass` **optional**; `rate_limit_raise`; `budget_tokens` + `model_tier` ∈ `{local_small, local_large}`; `prompt_route` ⊆ IR templates | ≤ **8h** (28800s) | optional |
| `break_glass` | All of `ir_elevated` + `model_tier` ∈ `{plane_ir, host_leased}` + emergency templates + tools ⊆ `packs/emergency-v0` (pack **may ship empty**) | ≤ **60m** (prefer **30m**) | **enterprise required**; **home** = owner + **required** auditor ticket |

### Ladder rules

- **Profile set** → `asks` are optional deltas ⊆ that profile’s max catalog. Still **non-empty** (axiom 4).
- **Profile null** → treat as **strict + named deltas**. Each ask must be a valid v0 kind and must fit the **strict** ceiling (baseline). Anything that needs `ir_elevated` / `break_glass` must name that profile.
- **Approve may drop asks** (amend down). Approve **must not** add asks, widen tools/templates, raise TTL, or lift `rails_profile` past what was requested.
- **`break_glass` forces** TTL clamp to the profile max (prefer 30m), an auditor ticket with `reason_code: break_glass`, `resolution.notes_redacted` on approve, and a **Phase B cooldown** before the next `break_glass` on that site (same idea as the notify door’s post-action cycle — not a second grant kind).
- **`net.quarantine_host` / `hypermesh.lease_stop` are not implied** by any profile. Add them only via explicit `tool_allowlist_add` when those actuators are ready — and only if the active profile max catalog includes them (today: not in `ir_elevated` defaults; only if listed in `packs/emergency-v0`).
- **Single grant TTL.** No per-ask TTL. `ttl_s` on resolve is the one clock.
- **`blast_radius` is `site` only** in v0.

`strict` is the resting policy. A grant does not leave the site in `ir_elevated` / `break_glass` after `active_until` or revoke.

## Object (`fyber.privilege_grant/v0`)

Docs-first resource. Extra keys fail validation. Shape reference: `examples/privilege_grant.example.json`.

```json
{
  "schema": "fyber.privilege_grant/v0",
  "grant_id": "<uuid>",
  "site_id": "net-tn-cottage",
  "incident_id": "<uuid>",
  "trace_id": "<uuid>",
  "requested_at": "2026-09-07T15:00:00Z",
  "requested_by": { "kind": "automation", "id": "brewnix-policy/v0" },
  "reason_redacted": "…",
  "asks": [
    {
      "kind": "tool_allowlist_add",
      "tools": ["health.restart_service", "notify.operator"]
    }
  ],
  "rails_profile_requested": "ir_elevated",
  "ttl_s_requested": 14400,
  "blast_radius": "site",
  "status": "proposed",
  "resolution": null,
  "active_until": null,
  "parent_grant_id": null,
  "ticket_id": null,
  "integrity": { "body_hash": "sha256:…" }
}
```

| Field | Required | Constraint |
|-------|----------|------------|
| `schema` | yes | const `fyber.privilege_grant/v0` |
| `grant_id` | yes (once minted / stored) | UUID. Assigned on propose (store). |
| `site_id` | yes | Same site scope as auditor tokens / envelopes. |
| `incident_id` | yes | UUID. Binds the elevation. Incident object is out of band (non-goal). |
| `trace_id` | yes | UUID (envelope / receipt cycle that proposed). |
| `requested_at` | yes | RFC3339 |
| `requested_by` | yes | `{ "kind", "id" }`. `kind`: `automation` \| `human`. `id`: 1–512. **Not** `model`. |
| `reason_redacted` | yes | 1–1000 chars; **no** secrets, payloads, or prompts. |
| `asks` | yes | Non-empty array of ask objects (below). |
| `rails_profile_requested` | yes | `strict` \| `ir_elevated` \| `break_glass` \| `null` |
| `ttl_s_requested` | yes | Positive int. Clamped / rejected against the profile max. |
| `blast_radius` | yes | const `site` |
| `status` | yes | `proposed` \| `approved` \| `denied` \| `timed_out` \| `revoked` \| `expired` |
| `resolution` | yes | object or `null` while `proposed` |
| `active_until` | yes | RFC3339 while `approved`; `null` otherwise |
| `parent_grant_id` | no | UUID of a prior grant this supersedes; `null` if none |
| `ticket_id` | no | Auditor `ticket_id` when a ticket is the approve UX. **Required** to mint `break_glass` (home and enterprise). |
| `integrity` | no | `{ "body_hash" }` — SHA-256 of canonical JSON with `integrity` omitted (same doctrine as `fyber.receipt/v0`). |

### `asks[]` items

Discriminated on `kind`. `additionalProperties: false` per item.

| `kind` | Required keys | Notes |
|--------|---------------|--------|
| `tool_allowlist_add` | `tools` | strings; each `ToolName` or `packs/<id>` |
| `rate_limit_raise` | `metric`, `limit` | `metric` = `blocks_per_hour`; `limit` ≥ 1 |
| `budget_tokens` | `max_tokens` | integer ≥ 1; sibling `model_tier` ask required |
| `model_tier` | `tier` | enum above; `plane_ir` / `host_leased` only on `break_glass` |
| `prompt_route` | `template_ids` | strings; ⊆ frozen registry ∩ profile template set |

Duplicate `kind` on one grant: reject (one ask per kind). `budget_tokens` without `model_tier` on the same grant: reject.

### `resolution` (after resolve)

```json
{
  "resolved_by": "<panopticon-subject>",
  "resolved_at": "2026-09-07T15:05:00Z",
  "ttl_s": 1800,
  "rails_profile": "break_glass",
  "notes_redacted": "…"
}
```

| Field | Required | Constraint |
|-------|----------|------------|
| `resolved_by` | yes | Human / dual-control subject id(s). Not an LLM. Enterprise `break_glass`: dual-control record (two humans). Home `break_glass`: site owner. |
| `resolved_at` | yes | RFC3339 |
| `ttl_s` | if `approved` | Minted TTL (≤ requested, ≤ profile max). `break_glass` clamped to ≤ 3600 (prefer 1800). |
| `rails_profile` | if `approved` | Minted profile; ⊆ requested ladder (may drop `break_glass` → `ir_elevated`, never the reverse). |
| `notes_redacted` | `break_glass` approve: **yes**; else optional | 1–1000; same redaction as `reason_redacted` |

`denied` / `timed_out` / `revoked` still set `resolved_by` + `resolved_at` (+ notes). `ttl_s` / `rails_profile` are `null` unless the grant was previously `approved` (revoke / expire keep the minted values for the audit trail).

`expired` may be set by a timeout worker (`resolved_by` like `panopticon:expiry`).

## Lifecycle

```
propose ──► proposed ──► resolve ──► approved ──► expire / revoke ──► expired / revoked
                 │                      │
                 ├─ denied              └─ site policy hot-reload for incident
                 └─ timed_out              executions via inference_iface + executor
                                           cite grant (annotate or future additive grant_id)
                                           then back to strict
```

1. **Propose.** Automation (or human) writes the grant body. Store assigns `grant_id`, `status: proposed`. Validate asks / profile / TTL / non-empty asks. No policy change yet.
2. **Optional auditor ticket UX.** `notify.operator` → `POST /v0/tickets` is how a human sees the ask. The ticket is inbox + resolution **intent** ([auditor API](fyber-auditor-api-v0.md)). `ticket_id` on the grant links them. One-shot companion execute still does **not** need a grant.
3. **Resolve.** Human or dual-control mints (`approved`), refuses (`denied`), or the propose TTL elapses (`timed_out`). Approve may drop asks; must not expand. `break_glass` approve requires ticket `reason_code: break_glass` + notes.
4. **Hot-reload.** Site policy for that `incident_id` applies the minted profile + asks until `active_until` (`resolved_at` + `ttl_s`).
5. **Execute.** Unchanged path: envelope proposals → policy → typed actuators → `fyber.receipt/v0`. Elevated execute is allowed only while the grant is `approved` and now < `active_until`. Cite the grant with `receipt.annotate` (or a later additive `grant_id` — **not** a v0 schema field).
6. **Expire / revoke.** Clock or human. Policy returns to **strict**. Elevated execute after `active_until` is **denied**.

`parent_grant_id` is how a later propose supersedes. It does not stack TTL or catalogs; the new grant is validated on its own.

### Offline

- A **cached approved** grant may finish its TTL if the plane drops. Site is source of truth for `active_until`.
- **Minting `break_glass` without the plane** is allowed **only** on **home** when local-owner approve is allowed. Enterprise `break_glass` requires dual-control on the plane. Still write the grant JSON + receipt locally. Queue the auditor ticket (`reason_code: break_glass`) as in the notify door.

## Ticket vs grant

| | `fyber.auditor.ticket/v0` | `fyber.privilege_grant/v0` |
|--|---------------------------|----------------------------|
| Job | One-shot held companion (`display.tool` + subject) | Time-bounded policy elevation for an incident |
| Empty body | Ticket always has one `display` | Empty `asks` → **refuse** (use a ticket) |
| Resolve | Intent; site apply / observe + ack | Mints (or denies) policy; site hot-reloads |
| `break_glass` | Ticket `reason_code: break_glass` is the required loud door | Grant profile + clamped TTL + notes |

The UI must not treat ticket `approved` as an elevation, or grant `approved` as a firewall apply.

## Acceptance tests

1. **Unknown ask rejected.** `kind: "mcp_allowlist"` (or any CUT / deferred / typo) → grant not stored; stays unminted.
2. **`break_glass` TTL over max rejected or clamped.** `ttl_s_requested: 7200` with `rails_profile_requested: break_glass` → reject **or** clamp to ≤ 3600 (prefer 1800) on approve. Never mint 2h break-glass.
3. **Empty asks rejected.** `asks: []` → refuse; caller should open a ticket for one-shot approve.
4. **Budget without `model_tier` rejected.** `budget_tokens` and no sibling `model_tier` → reject.
5. **Asks outside profile max rejected or amended down.** e.g. `model_tier: host_leased` on `ir_elevated`, or `tool_allowlist_add` of `hypermesh.lease_stop` when not in catalog → reject the propose **or** drop that ask on approve. Never mint past the profile.
6. **After `active_until`, elevated execute denied.** Policy is `strict` again. Baseline zero-LLM tools unchanged; IR-only tools / tiers / routes from the grant are not execute-eligible.
7. **LLM never in the executor.** Mint and execute paths are policy + human + typed actuators. No model `resolved_by`.
8. **No prompts in the grant body.** `reason_redacted` / `notes_redacted` only; no prompt text, packet payloads, or renter chat. Same doctrine as tickets / receipts.

## Change control

Additive clarifications to this doc are fine. A sixth ask kind, `blast_radius: org`, per-ask TTL, USD meters, `rails_profile` as an ask, or a `schemas/privilege_grant.v0.json` file need an explicit Chris amendment. Semantic changes to locked files under `schemas/` remain **v1** or a Chris-approved schema amendment.

Implement against this file. Do not wait on a schema file under `schemas/`.
