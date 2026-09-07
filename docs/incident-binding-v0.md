# fyber.incident binding v0

**Status:** working spec — **LOCKED 2026-09-07** (Chris; after pressure-test)  
**Schemas:** pin this repo (`fyber.inference_iface/v0`, `fyber.receipt/v0`, `fyber.feature_bundle/v0`, `fyber.common/v0`) — **no v0 schema break**  
**Site contract:** `fyber.incident/v0` (docs-first; **not** a file under `schemas/`)  
**Companion:** [fyber.privilege_grant v0](privilege-grant-v0.md) (requires `incident_id`) · [fyber.auditor API v0](fyber-auditor-api-v0.md) (one-shot ticket; empty-ask stays there) · [Zero-LLM OPNsense detect → block → receipt loop](zero-llm-opnsense-loop.md) · [Health-watch v0.1](health-watch-v0.md) · [Model triage v0](model-triage-v0.md) (model hold/propose opens/joins `security`; policy writes the record, not the LLM)

Goal: a **site-local overlay case** that groups multiple receipt cycles. A `trace_id` is one cycle. An `incident_id` is the multi-cycle case. Contain and `firewall.block_ip` **never** wait on this store. Grants **do** require an `incident_id`. Quiet observe / unscoped receipts do not.

**Home:** this repo holds the contract. Site open/close is source of truth. The plane **may** mirror later; it is not required to open. Site JSON Schemas stay locked; do not add `incident_id` to `schemas/` in this lock.

## Goal / non-goals

| Goal | Meaning |
|------|---------|
| Overlay case | Group related cycles (`receipt` / ticket / grant) under one `incident_id` without becoming an actuation gate. |
| Site SoT | Open and close happen on the site. Plane mirror is optional and later. |
| Two kinds only | `security` and `ops`. Never join across that line. |
| Two statuses only | `open` \| `closed`. No `contained` / `monitoring`. |
| Bind without schema amend | Side index now; optional additive `incident_id` on receipt/ticket later. |

| Non-goal | Why |
|----------|-----|
| Schema amend adding required `incident_id` | Separate lock. Receipt / ticket / envelope stay as they are. |
| Plane-required open | Offline site must open and close alone. |
| Replacing the receipt hash chain | Incident is not a chain object. Receipts stay the integrity spine. |
| EVE / payloads on the incident | Same redaction doctrine as receipts / tickets. `summary_redacted` only. |
| 24h security **join** window | Join is **4h** from `opened_at`. Auto-close quiet is the 24h clock. Do not conflate. |
| `contained` / `monitoring` states | Dropped. Status is `open` or `closed` only. |
| `supersedes_incident_id` | Deferred. Reopen after close = **new** `incident_id`. |
| JSON Schema under `schemas/` | Implement against this file. |

## Axioms

1. **`trace_id` ≠ `incident_id`.** Trace = one envelope / receipt cycle. Incident = the multi-cycle case.
2. **Site-local open/close is source of truth.** The plane may mirror later. Do not require the plane to open or close.
3. **Incident is an overlay — never a contain gate.** Do **not** gate `firewall.block_ip` / contain on incident-store success. If the incident write fails, **still act**. Attach the receipt later (side index or a later additive field).
4. **Grants require `incident_id`.** Propose / mint of `fyber.privilege_grant/v0` without one is rejected. Empty-ask one-shot tool approve stays on `fyber.auditor.ticket/v0` and does **not** require an incident.
5. **Not every receipt needs an incident.** Quiet observe / whitelist dedupe / unscoped cycles stay off the case.
6. **Binding is additive or side-index until a schema amend.** Optional later `incident_id` on receipt / ticket, **or** a site side index `incident_id → receipt_ids, ticket_ids, grant_ids`. Do not wait on `schemas/`.

## Object (`fyber.incident/v0`)

Docs-first resource. Extra keys fail validation. Shape reference: `examples/incident.example.json`.

```json
{
  "schema": "fyber.incident/v0",
  "incident_id": "<uuid>",
  "site_id": "net-tn-cottage",
  "kind": "security",
  "status": "open",
  "opened_at": "2026-09-07T14:00:00Z",
  "opened_by": { "kind": "rule", "id": "brewnix-rules/v0.1" },
  "severity": "critical",
  "summary_redacted": "…",
  "primary_subjects": [{ "kind": "ip", "value": "203.0.113.50" }],
  "closed_at": null,
  "close_reason": null,
  "flags": { "contain_applied": true, "grant_active": false },
  "links": {
    "opening_trace_id": "<uuid>",
    "opening_receipt_id": "<uuid>"
  }
}
```

| Field | Required | Constraint |
|-------|----------|------------|
| `schema` | yes | const `fyber.incident/v0` |
| `incident_id` | yes | UUID. Assigned on open. **Never** reuse after close. |
| `site_id` | yes | Same site scope as envelopes / grants / auditor tokens. |
| `kind` | yes | `security` \| `ops` only |
| `status` | yes | `open` \| `closed` only. **No** `contained` / `monitoring`. |
| `opened_at` | yes | RFC3339. Join window is measured from this instant. |
| `opened_by` | yes | `{ "kind", "id" }`. `kind`: `rule` \| `automation` \| `human`. `id`: 1–512. **Not** `model`. |
| `severity` | yes | `info` \| `low` \| `medium` \| `high` \| `critical` (same enum as site schemas). **Rolling max** of linked judgments while `open`. |
| `summary_redacted` | yes | 1–1000 chars; **no** secrets, payloads, prompts, or EVE. |
| `primary_subjects` | yes | Array, **cap 16**. Dominant subject is `[0]` (join key). |
| `closed_at` | yes | RFC3339 while `closed`; `null` while `open`. |
| `close_reason` | yes | `human` \| `auto_quiet` while `closed`; `null` while `open`. Other tokens are **not** v0. |
| `flags` | no | `{ "contain_applied", "grant_active" }` — booleans. **Derived OK** from the side index / grant store. |
| `links` | no | `{ "opening_trace_id", "opening_receipt_id" }` — UUIDs of the cycle that opened. |

`opened_by.kind` is the store actor (rule pack, policy automation, or human). The LLM never opens the record.

### `primary_subjects[]`

Discriminated on `kind`. Cap **16**. Join uses **`[0]` only** (dominant). Do not join on secondary subjects.

| Incident `kind` | Dominant subject | Notes |
|-----------------|------------------|-------|
| `security` | `{ "kind": "ip", "value": "<blocked IP>" }` (or `judgment.subjects[0]`) | `kind` + `value` must both match. Typically the blocked IP. |
| `ops` | `(node, unit)` **or** health signal class | **Not** an IP. Never join ops to a security IP case. |

Ops shapes (v0):

```json
{ "kind": "node_unit", "node": "opnsense-cottage", "unit": "suricata" }
```

```json
{ "kind": "health_class", "value": "health_disk" }
```

`node_unit` join matches both `node` and `unit`. `health_class` join matches `value` (e.g. `health_disk`, `health_wan`). Disk / WAN notifies that have no restart pair use `health_class`.

## Open / join triggers

| Event | Action | `kind` |
|-------|--------|--------|
| Critical `execute` contain (`firewall.block_ip` applied or attempted) | open or join | `security` |
| `hold_human` / `propose` + required `notify.operator` (IDS / contain; rules or [model-triage](model-triage-v0.md) — `opened_by.kind` is still not `model`) | open or join | `security` |
| Health notify **required** | open or join | `ops` |
| Grant / `break_glass` | **require** existing or **open** (usually `security`) | existing `kind`, or `security` when opening |
| Expiry (`brewnix-rules/expiry` / `firewall.unblock_ip`) | **inherit parent incident only**; never open new | parent `kind` |
| Quiet observe / whitelist dedupe / unscoped | **no** incident | — |

If the parent contain cycle never attached an incident (write failed, or observe-only), expiry still does **not** open one. Inherit or stay unscoped.

Grant / `break_glass` without a matching open incident: open one (usually `security`), then mint. Missing `incident_id` on the grant body → **reject** (grant spec + acceptance test 4). Empty `asks` is still a ticket, not a grant, and does not open an incident by itself.

## Join key

**All** of the following must hold. If any fails, do **not** join — open a new `incident_id` (or stay unscoped).

| Predicate | Rule |
|-----------|------|
| `site_id` | Equal. |
| `kind` | Equal. **Never** cross `security` ↔ `ops`. |
| `status` | Candidate is `open`. **Only open joins.** Closed never joins. |
| Dominant subject | **Security:** `kind` + `value` of `[0]` (blocked IP / `subjects[0]`). **Ops:** `(node, unit)` or health signal class — **not** IP. |
| `opened_at` within `JOIN_WINDOW` | Measured from the candidate’s `opened_at`, **not** last activity. **Security = 4h. Ops = 1h** (locked default). |
| Closed | Reopen after `closed` = **new** `incident_id`. No join across closed. `supersedes_incident_id` is deferred. |

An incident that is still `open` but past `JOIN_WINDOW` does **not** receive joins. The caller opens a new id. Two open incidents for the same subject can coexist when the first aged past the window; close clocks stay independent.

**Ops window vs service clear.** Locked default for tests is **1h** from `opened_at`. An implementation **may** keep joining the same **open** ops incident until the health signal clears (Suricata `running`, disk below threshold, WAN `up`) even if that is longer than 1h — document the choice in the site engine. Do not silently treat “until clear” as a 24h security-style join.

Do **not** refuse a join because the new cycle’s severity is lower than the case. Severity is a rolling max, not a join key.

## Close

| Rule | Behavior |
|------|----------|
| Active grant | **Refuse** close while any linked `fyber.privilege_grant/v0` is `approved` and now < `active_until`. Revoke or expire the grant first. |
| Human close | OK if no active grant. Sets `status: "closed"`, `close_reason: "human"`, `closed_at` now. |
| Auto-close | No **new** receipts linked to **this** `incident_id` for the quiet period **and** no active grant. Security quiet = **24h**. Ops quiet = **2h after health clear**. `close_reason: "auto_quiet"`. |
| Quiet ≠ site-wide silence | Quiet is only “no linked receipts for this id”. Other IPs / other kinds / unscoped observe do not reset this clock. Site-wide alert mute is out of scope. |
| Close race | Set `closed` **then** reject new links to that id. The caller opens a **new** incident. Do not reopen. |
| Break-glass Phase B | Cooldown keys off `closed_at` of the incident that **held** the grant — not grant `active_until` alone, not a site-wide mute. See [privilege-grant](privilege-grant-v0.md). |
| Tickets | Remain historical. Keep `ticket_id` on the side index (or a later additive `incident_id` on the ticket). Closing the case does not delete tickets. |

`flags.grant_active` must be false (or derived false) before close succeeds.

## Severity

Rolling **max** of linked judgments while `open`. A `medium` join onto a `critical` case leaves `severity: "critical"`. A later `critical` join onto a `high` case raises it. Do not freeze severity at open. Do not refuse joins for lower severity. After close, severity is the max observed while it was open (historical; do not keep rolling).

## Binding (until schema amend)

Do **not** add required `incident_id` to `fyber.receipt/v0` / `fyber.auditor.ticket/v0` in this lock.

**Now:** a site side index, e.g. `/var/lib/brewnix/incident_index.jsonl`:

```json
{
  "incident_id": "<uuid>",
  "receipt_ids": ["<uuid>", "…"],
  "ticket_ids": ["<uuid>"],
  "grant_ids": ["<uuid>"]
}
```

Attach after the fact when the overlay write failed but contain already applied (axiom 3).

**Later (additive, optional, behind a flag):** `incident_id` on receipt and/or ticket. Still not required. Quiet observe / unscoped stay omitted. That field is a Chris amendment to the site schemas — not this lock.

Cite a grant with `receipt.annotate` (or the side index). Same doctrine as [privilege-grant](privilege-grant-v0.md).

## Offline

Open and close on the site with the plane down. Same incident JSON either way. Queue any plane mirror (and any auditor ticket / grant propose) as in the notify door. Cached `grant_active` follows the grant spec (site is SoT for `active_until`). Auto-close and human close do not wait on Panopticon.

## Acceptance tests

1. **Critical block opens/joins a security incident; contain still works if the incident write fails.** Synthetic `port_scan_burst` → `execute` → alias add. Incident open/join is attempted. Inject an incident-store failure → `firewall.block_ip` still `applied` (or still attempted) and the receipt still chains. Attach the receipt later via the side index.
2. **Two bursts, same IP, within 4h → one incident; after close → new id.** Second critical/hold cycle for `203.0.113.50` inside 4h of `opened_at` joins. After `closed`, a third burst gets a **new** `incident_id`. Closed never joins.
3. **Ops Suricata-down does not join a security IP incident.** `health.restart_service` + required notify (`brewnix-rules/health-v0.1`) opens or joins `kind: "ops"` with `(node, unit)` or `health_class`. It must not land on the open security case for `203.0.113.50`.
4. **Grant without `incident_id` rejected.** `fyber.privilege_grant/v0` missing / empty / null `incident_id` is not stored and not minted. One-shot empty-ask approve still goes to the auditor ticket and does not require an incident.
5. **Expiry inherits; no new incident.** `brewnix-rules/expiry` unblock cites the parent contain incident (side index / inherit). It never opens a new `incident_id`. If the parent had none, expiry stays unscoped.
6. **Close rejected while a grant is active.** `approved` grant with `now < active_until` linked to this id → human close and auto-close both refuse. After revoke / expire, human close is allowed; auto-close may then fire if quiet is also satisfied.
7. **Observe unscoped.** Whitelist / already-blocked dedupe / quiet observe write a receipt and **do not** open or join an incident.
8. **Offline open/close on site.** `plane_reachable: false` → site still opens on a contain/notify trigger, still refuses close while a (cached) grant is active, still auto-closes after quiet. Plane mirror is optional later.

## Change control

Additive clarifications to this doc are fine. A required `incident_id` on `schemas/receipt.v0.json` (or tickets), a plane-required open, `contained` / `monitoring` statuses, a 24h **join** window, `supersedes_incident_id`, EVE on the object, or a `schemas/incident.v0.json` file need an explicit Chris amendment. Semantic changes to locked files under `schemas/` remain **v1** or a Chris-approved schema amendment.

Implement against this file. Do not wait on a schema file under `schemas/`.
