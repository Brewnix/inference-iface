# fyber.auditor API v0

**Status:** working spec — **LOCKED 2026-09-07** (Chris)  
**Schemas:** pin this repo (`fyber.inference_iface/v0`, `fyber.receipt/v0`, `fyber.feature_bundle/v0`, `fyber.common/v0`) — **no v0 schema break**  
**Plane contract:** `fyber.auditor.ticket/v0` (HTTP; **not** a file under `schemas/`)  
**Companion:** [notify.operator ↔ shared auditor door](notify-operator-audit-door.md) (site half) · [Zero-LLM OPNsense detect → block → receipt loop](zero-llm-opnsense-loop.md)

Goal: a **shared prompt-router / auditor API** on the Hypermesh / Panopticon plane. Sites already write a local `fyber.receipt/v0` and POST a typed `notify.operator`. This doc is the plane door those POSTs hit. The plane is a human inbox + resolution **intent**. Actuation truth stays on the site receipt.

**Home:** this repo holds the contract. **Panopticon** (or a dedicated service it fronts) implements. Not hyperme.sh the marketing site. Site JSON Schemas stay locked; do not add ticket fields to `schemas/`.

## Goal / non-goals

| Goal | Meaning |
|------|---------|
| Shared plane service | Same family as a gateway in front of customer-owned models for authorized security / IR work + an exportable receipt. Reusable across products. |
| Four verbs only | Create (idempotent), get by id, resolve, site ack. |
| Token scope | Panopticon **machine/site token**. Tickets are scoped by `site_id` **from the token**. |
| Intent vs actuation | Resolve is plane intent. Site apply / observe writes the receipt, then acks. UI must not claim blocked from resolve alone. |

| Non-goal (CUT from earlier draft) | Why |
|-----------------------------------|-----|
| `POST /v0/route` (or any `/route`) | Router / multi-tool dispatch is not this door. |
| `GET /v0/tickets` collection / inbox | No list. Caller already has `ticket_id` from create (or a later annotate). Plane UI may use a private store; it is not this API. |
| Per-ticket `callback` URL | Site polls `GET /v0/tickets/{id}`. |
| Args digests on the ticket | Digests live on the site receipt (`inputs_digest` / `features_digest` / `integrity.body_hash`). |
| Free-form multi-tool bags | `display` is one companion tool + one subject + optional `ttl_s`. |
| Site mTLS in v0 | Machine/site token only. |
| hyperme.sh the website | May render tickets later. The **API** is the door. |
| Rewriting site rules or executing tools | Auditor never calls OPNsense, never writes `rules/*.yaml`. Phase B rule bumps stay on-site. |

## Axioms

1. **Site receipt is actuation source of truth.** A block / observe / hold happened only when `fyber.receipt/v0` says so (`execution[]`, `policy.decision`, `human.resolution`).
2. **Plane is inbox + resolution intent.** `approved` / `denied` / `timed_out` / `amended` on the ticket is what a human (or timeout worker) decided. It is not an apply.
3. **Offline queue on site.** Local receipt always. If `plane_reachable: false` or POST fails, durable-queue the create (see notify door). Same ticket JSON either way.
4. **Auditor never rewrites site rules or executes tools.** No alias mutation, no SID toggles, no pack bumps from the plane process.
5. **Digests / redaction doctrine.** No prompts, packet payloads, secrets, or renter chat. Ticket text is `text_redacted` / `notes_redacted` only. Do not add args digests to this contract.
6. **Phase B rule bumps stay on-site.** `amended` may carry an allowlisted `patch` for the **held** companion tool. Changing future detectors is a new site cycle with `actor.kind: "rule"` and a new `actor.id` (notify door). The plane does not ship a new rule pack.

## Auth

`Authorization: Bearer <panopticon-machine-site-token>`

| Rule | Behavior |
|------|----------|
| Unknown / invalid / expired token | **Reject** (no ticket leak). |
| Token `site_id` | Sole scope. Body `site_id` must equal the token's site; else reject. |
| Cross-site `{id}` | **404** (do not disclose existence). |
| Site mTLS | **Not** v0. |

## Endpoints (v0 ONLY)

| Method | Path | Who | Role |
|--------|------|-----|------|
| `POST` | `/v0/tickets` | site | Create (idempotent upsert) |
| `GET` | `/v0/tickets/{id}` | site or plane UI | Read one ticket |
| `POST` | `/v0/tickets/{id}/resolve` | plane human / timeout worker | Set resolution **intent** |
| `POST` | `/v0/tickets/{id}/ack` | site | Site has written the apply / observe receipt |

`{id}` is `ticket_id` (UUID). No other `/v0` paths. `GET /v0/tickets` without an id, and any `/route`, are **not** in contract — treat as missing.

## `POST /v0/tickets` — create

Idempotent. Return the **same** `ticket_id` on replay.

**Upsert key**

| When | Natural key |
|------|-------------|
| `receipt_id` is a UUID | `(site_id, receipt_id)` |
| `receipt_id` is `null` | `(site_id, trace_id, reason_code)` |

Body (`fyber.auditor.ticket/v0`):

```json
{
  "schema": "fyber.auditor.ticket/v0",
  "site_id": "net-tn-cottage",
  "trace_id": "<uuid>",
  "receipt_id": "<uuid|null>",
  "held_call_id": "<uuid>",
  "reason_code": "rate_limit_hold",
  "severity": "high",
  "text_redacted": "…",
  "display": {
    "tool": "firewall.block_ip",
    "subject": { "kind": "ip", "value": "203.0.113.50" },
    "ttl_s": 86400
  }
}
```

| Field | Required | Constraint |
|-------|----------|------------|
| `schema` | yes | const `fyber.auditor.ticket/v0` |
| `site_id` | yes | must match token site |
| `trace_id` | yes | UUID (envelope / receipt cycle) |
| `receipt_id` | yes | UUID of the **held** site receipt, or `null` if not yet known |
| `held_call_id` | yes | UUID of the **companion** tool call (not the `notify.operator` `call_id`) |
| `reason_code` | yes | `^[a-z][a-z0-9_]{0,63}$` — e.g. `rate_limit_hold`, `propose_needs_ack`, `post_action_audit` |
| `severity` | yes | `info` \| `low` \| `medium` \| `high` \| `critical` (same enum as site schemas) |
| `text_redacted` | yes | 1–1000 chars; **no** secrets, payloads, or prompts |
| `display.tool` | yes | one allowlisted companion tool (v0 loop: `firewall.block_ip` / `firewall.unblock_ip`) |
| `display.subject` | yes | `{ "kind", "value" }` — `kind` as site `SubjectKind` |
| `display.ttl_s` | no | held TTL; 1–604800 when present |

No `callback`, no args object, no digest fields, no extra tool list. Extra keys fail validation.

Response: ticket resource (`status: "open"`, resolution fields + `site_acked_receipt_id` all `null`). Shape reference: `examples/auditor.ticket.example.json`.

### Site → plane mapping

From the notify door ToolCall / envelope / receipt (until additive site fields exist):

| Ticket field | Source |
|--------------|--------|
| `site_id` / `trace_id` | envelope / receipt |
| `receipt_id` | receipt; else parse / copy from `text_redacted` / `execution.effect` / `receipt.annotate` |
| `held_call_id` | companion `firewall.*` `call_id` |
| `reason_code` / `severity` / `text_redacted` | `notify.operator` |
| `display.tool` / `ttl_s` | companion ToolCall |
| `display.subject` | companion args (e.g. `ip`) or `judgment.subjects[]` |

## `GET /v0/tickets/{id}`

Returns the ticket resource.

| Field | Values / notes |
|-------|----------------|
| `status` | `open` \| `resolved` \| `acked` |
| `resolution` | `approved` \| `denied` \| `timed_out` \| `amended`, or `null` while `open` |
| `resolved_by` | Panopticon subject, or `null` while `open` |
| `resolved_at` | RFC3339, or `null` while `open` |
| `patch` | object or `null` — only meaningful when `resolution` is `amended` |
| `notes_redacted` | string or `null` |
| `site_acked_receipt_id` | **`null` until site ack** — then the apply / observe receipt UUID |

Create fields (`schema`, `ticket_id`, `site_id`, `trace_id`, `receipt_id`, `held_call_id`, `reason_code`, `severity`, `text_redacted`, `display`) echo from create. `receipt_id` here is the **held** receipt from create, not the ack receipt.

## `POST /v0/tickets/{id}/resolve`

Plane-only. Valid from `open`. Replay of an **identical** resolve returns the same ticket. Conflicting resolve after `resolved` / `acked` is rejected.

```json
{
  "resolution": "approved|denied|timed_out|amended",
  "resolved_by": "<panopticon-subject>",
  "patch": { "ttl_s": 3600 },
  "notes_redacted": "…"
}
```

| Field | Required | Constraint |
|-------|----------|------------|
| `resolution` | yes | `approved` \| `denied` \| `timed_out` \| `amended` (same enum as `fyber.receipt/v0` `human.resolution`) |
| `resolved_by` | yes | Panopticon subject id (human or timeout worker) |
| `patch` | only if `amended` | object; **omit or `null` otherwise**. Keys **allowlisted per `display.tool`** |
| `notes_redacted` | no | 1–1000 when present; same redaction doctrine as `text_redacted` |

**`patch` allowlist (v0)**

| `display.tool` | Allowed keys |
|----------------|--------------|
| `firewall.block_ip` | `ttl_s` only (integer 1–604800) |
| anything else | **none** — any `patch` is rejected |

`resolution` ≠ `amended` + `patch` present → reject.  
`resolution` = `amended` + missing / empty / unknown key / bad type → reject.

Sets `status: "resolved"`. `site_acked_receipt_id` stays `null`. Site still has not applied.

Site behavior after GET (notify door): `approved` → re-execute the held companion. `amended` → `receipt.annotate`; apply only allowlisted `patch` keys (do not blindly replay original args); Phase B pack bumps stay on-site. `denied` / `timed_out` → `observe`. Write a **new** receipt; `parent_id` = held receipt. Then ack.

## `POST /v0/tickets/{id}/ack`

Site-only. Valid from `resolved`. Plane resolution is **intent until this call**.

```json
{
  "receipt_id": "<uuid of apply/observe receipt>"
}
```

`receipt_id` is the **new** site receipt (apply or observe), not the held `receipt_id` from create. Sets `status: "acked"` and `site_acked_receipt_id` to that UUID. Replay with the same ack `receipt_id` returns the same ticket. Ack while `open`, or a different `receipt_id` after acked, is rejected.

## Status machine

```
POST /tickets ──► open ──► POST …/resolve ──► resolved ──► POST …/ack ──► acked
```

| `status` | Plane | Site | UI |
|----------|-------|------|-----|
| `open` | in inbox | waiting | not blocked |
| `resolved` | intent recorded | must GET, then apply / observe | **not** blocked — pending site (or **timed_out waiting**) |
| `acked` | done | actuation SoT is `site_acked_receipt_id` | may show blocked **only** if that receipt `execution` applied a block |

## UI rule

The UI **must not** claim the subject is blocked (or otherwise actuated) until the site acks, and the acked receipt shows an apply — **or** the operator is looking at a **timed_out waiting** state (resolved `timed_out`, ack not yet in / observe pending). Resolve `approved` is not a block.

## Acceptance tests

1. **Idempotent create.** Two `POST /v0/tickets` with the same upsert key return the same `ticket_id`. Different `receipt_id` (when set) → different ticket. Key is `(site_id, receipt_id)` when `receipt_id` is set; else `(site_id, trace_id, reason_code)`.
2. **create → resolve → site ack.** `open` → `resolved` (`site_acked_receipt_id` still `null`) → `acked` with `site_acked_receipt_id` = ack body `receipt_id`.
3. **Unknown token rejected.** Missing / garbage Bearer → no ticket created or returned.
4. **Amended with bad patch rejected.** `resolution: "amended"` + `{ "ttl_s": 3600, "alias": "x" }` (or `ip`, or patch on a tool with no allowlist, or patch when `resolution` is `approved`) → reject; ticket stays `open`.
5. **No `/route`.** `POST /v0/route` (and `GET /v0/tickets` collection) are not implemented.
6. **UI must not claim blocked until site ack** (or **timed_out waiting**). After `approved` resolve, before ack, UI / copy is pending-intent, not "blocked". After ack of an apply receipt, blocked is allowed. `timed_out` + still waiting on site ack is "timed_out waiting", not blocked.

## Change control

Additive clarifications to this doc are fine. New paths (`/route`, list/inbox), a `callback` field, args digests, multi-tool bags, or site-schema edits need an explicit Chris amendment. Semantic changes to locked files under `schemas/` remain **v1** or a Chris-approved schema amendment.

Implement against this file. Do not wait on a `schemas/auditor.ticket.v0.json`.
