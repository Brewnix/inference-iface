# Fyber inference interface + receipt — v0 (LOCKED)

Working contract locked **2026-09-06** by Chris:

- `fyber.inference_iface/v0` — judgment + typed tool **proposals** only
- `fyber.receipt/v0` — digests, policy decision, execution effects, hash chain
- LLM **never** mutates firewall/host state; policy engine + APIs/scripts execute
- No prompts, packet payloads, or renter chat in receipts
- Offline-first: same JSON with Panopticon unreachable

Breaking changes require an explicit **v1** or Chris-approved amendment. Additive optional fields may ship behind a flag without bumping the `schema` const.

## Layout

```
README.md
LOCK.md
schemas/
  common.v0.json           # $defs: Actor, Judgment, ToolCall, …
  feature_bundle.v0.json   # redacted model inputs
  inference_iface.v0.json  # envelope
  receipt.v0.json          # receipt
examples/
  feature_bundle.example.json
  envelope.example.json
  receipt.example.json
  notify.operator.example.json
  auditor.ticket.example.json
  expiry.unblock.example.json
  health.watch.example.json
  privilege_grant.example.json
  incident.example.json
  model-triage.envelope.example.json
  hypermesh-preempt.example.json
  sociacl-ir-binding.example.json
docs/
  zero-llm-opnsense-loop.md
  expiry-unblock-loop.md
  health-watch-v0.md
  notify-operator-audit-door.md
  fyber-auditor-api-v0.md
  privilege-grant-v0.md
  incident-binding-v0.md
  model-triage-v0.md
  hypermesh-preempt-v0.md
  sociacl-ir-binding-v0.md
```

Canonical home: this repo (`Brewnix/inference-iface`). Downstream consumers (Brewnix policy executor, IR door, Hypermesh host preempt) should pin a release or submodule path rather than forking the schemas.

## `$id` URIs

| File | `$id` |
|------|-------|
| common | `https://fyberlabs.com/schemas/fyber.common/v0` |
| feature bundle | `https://fyberlabs.com/schemas/fyber.feature_bundle/v0` |
| envelope | `https://fyberlabs.com/schemas/fyber.inference_iface/v0` |
| receipt | `https://fyberlabs.com/schemas/fyber.receipt/v0` |

Cross-refs use those `$id`s. For local validation, configure your schema loader to map each `$id` to the sibling file under `schemas/` (do not require network fetch).

## Tool catalog (v0 allowlist)

`firewall.block_ip` · `firewall.unblock_ip` · `net.quarantine_host` · `ids.suricata_pass` · `health.restart_service` · `health.set_nvpmodel` · `hypermesh.lease_stop` · `hypermesh.sell_pause` · `notify.operator` · `receipt.annotate`

Pre-provision OPNsense alias + block rule; mutate alias membership only. No `shell.exec`.

## Validation notes

1. Subjects in `judgment` / tool args that refer to IPs should appear in the feature bundle when the actor is `model` (enforced by policy, not only JSON Schema).
2. `integrity.body_hash`: hash canonical JSON of the receipt with `integrity` omitted (or with `body_hash`/`sig` set null — pick one algorithm and stick to it in the executor).
3. `inputs_digest` / `features_digest`: SHA-256 of canonical feature-bundle JSON, encoded `sha256:` + 64 hex chars.

## Non-goals (v0)

Hypermesh renter chat schema · SociACL primitive crate (this repo binds only) · SaaS-specific envelopes · raw Suricata EVE as model context.

## Specs

- [Zero-LLM OPNsense detect → block → receipt loop](docs/zero-llm-opnsense-loop.md) — first executor path (`actor.kind: rule`)
- [Expiry → unblock → receipt](docs/expiry-unblock-loop.md) — TTL ledger + `brewnix-rules/expiry` so `ai_autoblock` members do not rot
- [Health-watch v0.1](docs/health-watch-v0.md) — sibling non-IDS pack from `feature_bundle.health` (propose + notify; no auto-execute)
- [notify.operator ↔ shared auditor door](docs/notify-operator-audit-door.md) — site half of the plane auditor door; required on `hold_human` / high `propose`
- [fyber.auditor API v0](docs/fyber-auditor-api-v0.md) — locked plane ticket contract (`fyber.auditor.ticket/v0`); Panopticon implements
- [fyber.privilege_grant v0](docs/privilege-grant-v0.md) — locked plane/site grant (`fyber.privilege_grant/v0`); automations propose, human mints; **not** a one-shot ticket
- [fyber.incident binding v0](docs/incident-binding-v0.md) — locked overlay case (`fyber.incident/v0`); site open/close SoT; never gates contain; side index until schema amend
- [Model triage v0](docs/model-triage-v0.md) — locked judge-only model path (`actor.kind: model`); `rules_primary` default; never in the executor; no `schemas/` change
- [Hypermesh preempt v0](docs/hypermesh-preempt-v0.md) — locked Host preempt (`hypermesh.lease_stop` / `hypermesh.sell_pause`); Brewnix proposes, Host executes; tools **never** implied by `rails_profile`; Host inventory may lag; no `schemas/` change
- [SociACL IR binding v0](docs/sociacl-ir-binding-v0.md) — locked Brewnix binding of SociACL Check + `delegate` / `see` onto `:ir` / `:host`; crate remains SoT; grant ≠ delegate; no `schemas/` change
