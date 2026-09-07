# notify.operator ↔ shared auditor door

**Status:** working spec (2026-09-06)  
**Schemas:** pin this repo (`fyber.inference_iface/v0`, `fyber.receipt/v0`, `fyber.feature_bundle/v0`, `fyber.common/v0`) — **no v0 schema break**  
**Companion:** [Zero-LLM OPNsense detect → block → receipt loop](zero-llm-opnsense-loop.md) · [fyber.auditor API v0](fyber-auditor-api-v0.md) (plane ticket: `fyber.auditor.ticket/v0`)

Goal: when policy cannot auto-execute (rate-limit hold, non-auto propose) or after an action needs review, the site posts a **typed** `notify.operator` to a **shared auditor API** in the Hypermesh / Panopticon plane. The site still writes a local `fyber.receipt/v0` first. The plane is a door, not a prerequisite.

Plane half (create / get / resolve / ack): [fyber.auditor API v0](fyber-auditor-api-v0.md). `notify.operator` is the site actuator that POSTs `fyber.auditor.ticket/v0`. The ticket is inbox + resolution **intent** until the site acks with an apply / observe receipt.

## Goal / non-goals

| Goal | Meaning |
|------|---------|
| Shared plane service | Prompt-router / auditor **API** reusable across products (same family as a gateway in front of customer-owned models for authorized security / IR work + an exportable receipt). Prior bet: `FyberLabs/aimonitoring-doc`. That repo is **not** this door's implementation. |
| Two phases | (A) during incident holds / non-auto propose; (B) after-action audit / review and changing future rules. |
| Offline-first | Local receipt always. Queue notify when `plane_reachable: false`. Same JSON either way. |
| Locked tool args | `ArgsNotifyOperator` remains only `channel`, `severity`, `text_redacted`. |

| Non-goal | Why |
|----------|-----|
| hyperme.sh the marketing site | May UI tickets later. The **API** is the door. |
| New v0 fields (`receipt_id`, `ticket_id` on args) | Carry `receipt_id` in `text_redacted` and/or `execution.effect` and/or `receipt.annotate` until an additive amendment. |
| Expanding auto-execute tools | Zero-LLM loop auto path stays `firewall.block_ip` / `firewall.unblock_ip` only. Notify is a **separate** actuator. |
| Claiming a website channel | Policy-allowlisted channel id is `fyber.auditor` (service id, not a URL). |

## Locked tool shape (`notify.operator`)

From `schemas/common.v0.json` → `$defs/ArgsNotifyOperator` + `ToolCall`. Do not extend.

| Field | Required | Constraint |
|-------|----------|------------|
| `call_id` | yes | UUID |
| `tool` | yes | `notify.operator` |
| `mode` | yes | `dry_run` \| `propose` \| `execute` — policy **executes** notify on hold / ack paths even when the companion firewall tool stays `propose` |
| `reason_code` | yes | `^[a-z][a-z0-9_]{0,63}$` (e.g. `rate_limit_hold`, `propose_needs_ack`, `post_action_audit`) |
| `ttl_s` | no | unused for notify in v0 |
| `args.channel` | yes | string 1–64; policy allowlist includes **`fyber.auditor`** |
| `args.severity` | yes | `info` \| `low` \| `medium` \| `high` \| `critical` |
| `args.text_redacted` | yes | string 1–1000; **no** secrets, payloads, or prompts |

`additionalProperties: false` on args. Extra keys fail schema validation.

## Two phases

| Phase | When | What crosses the door | What stays local |
|-------|------|------------------------|------------------|
| **A — incident** | Policy `hold_human` (rate limit) or `propose` that needs ack (v0.1: severity `high`) | `notify.operator` → auditor API (or durable queue) | `fyber.receipt/v0` with `human.required: true`; companion firewall tool **not** auto-executed |
| **B — post-action** | After an `execute` (or later IR review), optional rule-tuning | `notify.operator` with `reason_code: post_action_audit`; later `receipt.annotate` | Receipt already chained; rule bumps are a new cycle with `actor.kind: "rule"` and a new `actor.id` |

Phase B is **optional**. Phase A notify is **required** on the hooks below.

## Policy hooks (`brewnix-policy/v0`)

| Hook | Policy decision | Notify | Envelope / receipt flags |
|------|-----------------|--------|--------------------------|
| `rate_limit_hold` | `hold_human` | **required** `notify.operator` (`channel: fyber.auditor`) | `needs_human: true`; `human.required: true` |
| `propose_needs_ack` | `propose` (v0.1: severity `high`) | **required** `notify.operator` | same |
| `post_action_audit` | after a prior `execute` / review | optional | `needs_human: true` if a human must ack the review; else annotate-only |

Policy still reject-unknown-tools / whitelist / dedupe as in the companion loop. These hooks **do not** add firewall tools to the auto-execute set.

If policy would emit `hold_human` or ack-required `propose` **without** a valid `notify.operator` proposal, that is a policy bug: do not execute the companion tool; still write a receipt (`observe` or `hold_human`) recording the miss.

## Example ToolCall

```json
{
  "call_id": "550e8400-e29b-41d4-a716-446655440002",
  "tool": "notify.operator",
  "mode": "execute",
  "reason_code": "rate_limit_hold",
  "args": {
    "channel": "fyber.auditor",
    "severity": "high",
    "text_redacted": "hold_human site=net-tn-cottage receipt_id=550e8400-e29b-41d4-a716-446655440010 ip=203.0.113.50 rule=ssh_brute"
  }
}
```

Shape reference: `examples/notify.operator.example.json` (full envelope).

Until additive fields exist, put `receipt_id` in `text_redacted` (as above) and/or copy it into `execution.effect` and/or a follow-up `receipt.annotate`.

## Executor (notify actuator)

Not OPNsense. A site-local notify actuator calls the **shared auditor API** (Hypermesh / Panopticon plane).

1. Validate `channel` ∈ policy allowlist (`fyber.auditor` required on Phase A hooks).
2. If `plane_reachable: true` → `POST /v0/tickets` per [fyber.auditor API v0](fyber-auditor-api-v0.md) (Panopticon machine/site token). Record `execution.status: "applied"`.
3. If plane down or the POST fails → **durable queue**; record `execution.status: "applied"` with `effect.queued: true` (or `failed` only if the queue itself cannot persist). Never drop the local receipt.
4. Append `execution[]`:

```json
{
  "call_id": "550e8400-e29b-41d4-a716-446655440002",
  "tool": "notify.operator",
  "status": "applied",
  "executor": "auditor-api@site",
  "effect": {
    "channel": "fyber.auditor",
    "ticket_id": null,
    "queued": false
  },
  "error": null
}
```

`effect` is an open object on `fyber.receipt/v0` — keep it small and non-secret. Suggested keys: `channel`, `ticket_id` (string or `null` until the plane assigns one), `queued` (boolean). Optional: `receipt_id` echo.

Drain the queue when the plane returns; do not rewrite the original receipt. A later receipt (or `receipt.annotate`) may record `ticket_id` once known.

## Human resolution

`fyber.receipt/v0` → `human.resolution` enum (locked):

| Resolution | Site behavior |
|------------|----------------|
| `approved` | Re-execute the **held / proposed** companion tool (still only `firewall.block_ip` / `firewall.unblock_ip` on the zero-LLM auto path). New receipt; `parent_id` = held receipt. |
| `denied` | `observe` — do not execute the companion tool. Annotate if useful. |
| `timed_out` | `observe` — same as denied for actuation. Annotate timeout. |
| `amended` | Do not blindly re-execute the original args. `receipt.annotate` the change. Next detect cycle may use a bumped rule pack. |

Rule bumps: `actor.kind: "rule"`, `actor.id` like `brewnix-rules/v0.2` (or `brewnix-rules/v0.1+amend-<short>`). Do not mutate a past receipt's actor. Phase B auditor review is how a human changes **future** rules, not the locked schemas.

Pending: `human.required: true`, `resolved_by` / `resolved_at` / `resolution` all `null`.

Plane `POST /v0/tickets/{id}/resolve` is **intent** only. After the site writes the apply / observe receipt, `POST /v0/tickets/{id}/ack` ([fyber.auditor API v0](fyber-auditor-api-v0.md)). UI must not claim blocked until that ack (or a timed_out waiting state).

## Acceptance tests

1. JSON Schema validate a `notify.operator` ToolCall / envelope (`examples/notify.operator.example.json`); extra args keys fail; `channel` / `severity` / `text_redacted` only.
2. Rate-limit path (`hold_human`) **requires** `notify.operator` with `channel: fyber.auditor`; `needs_human` and `human.required` are `true`; companion block is **not** applied.
3. Severity `high` → `propose` **requires** the same notify; no auto `firewall.block_ip`.
4. `plane_reachable: false` → receipt still appended locally; notify `effect.queued: true`; queue drains later without rewriting the original receipt.
5. Resolution `approved` → re-execute companion tool + child receipt; `denied` / `timed_out` → `observe`; `amended` → annotate only + later cycle may use a new `actor.id`.
6. Unknown / non-allowlisted `channel` rejected by policy; `fyber.auditor` is a service id, not a website; auto-execute tool set unchanged (`firewall.block_ip` / `firewall.unblock_ip` only).

## Change control

Additive clarifications to this doc are fine. New required args, renaming `fyber.auditor`, or putting SaaS / website URLs in `channel` need an explicit amendment. Semantic changes to locked schemas remain **v1** or a Chris-approved schema amendment.
