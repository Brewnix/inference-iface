# Health-watch v0.1 (zero-LLM)

**Status:** working spec (2026-09-07)  
**Rules pack:** `brewnix-rules/health-v0.1`  
**Schemas:** pin this repo (`fyber.inference_iface/v0`, `fyber.receipt/v0`, `fyber.feature_bundle/v0`, `fyber.common/v0`) — **no v0 schema break**  
**Companion:** [Zero-LLM OPNsense detect → block → receipt loop](zero-llm-opnsense-loop.md) · [notify.operator ↔ shared auditor door](notify-operator-audit-door.md) · [fyber.auditor API v0](fyber-auditor-api-v0.md)

Goal: a **zero-LLM** health watch from `feature_bundle.health`, using the same envelopes and receipts as the IDS contain loop. Conservative policy. This is a **sibling non-IDS pack** — it does not emit `firewall.block_ip` / `firewall.unblock_ip`.

## Components (site box)

| Piece | Role |
|-------|------|
| **Sensor** | Same redacted `fyber.feature_bundle/v0` as the companion loop (`health.cpu` / `disk` / `wan_gateways` / `suricata`) |
| **Rules engine** | `actor.id: "brewnix-rules/health-v0.1"` emits typed `proposals` |
| **Policy** | Conservative: **auto-execute off** for health in v0.1 → `observe` \| `propose` |
| **Notify actuator** | Required `notify.operator` → `fyber.auditor` (reuse [slice 3](notify-operator-audit-door.md)) |
| **Receipt store** | Same `fyber.receipt/v0` JSONL + hash chain |

Health does **not** share the IDS auto-execute set. [Expiry unblock](expiry-unblock-loop.md) is the other sibling (contain TTL); it is also not this pack.

### Envelope (`fyber.inference_iface/v0`)

```json
"actor": { "kind": "rule", "id": "brewnix-rules/health-v0.1", "purpose": "triage" }
```

Envelope must include the same required fields as the companion loop. `inputs_digest` = `features_digest` of the cycle's bundle. `judgment.classes` includes **`health`** (locked `ThreatClass`). `needs_human: true` whenever notify is required.

Shape reference: `examples/health.watch.example.json`.

## Triggers (from `feature_bundle.health`)

Locked bundle fields (`schemas/feature_bundle.v0.json`): `cpu` / `disk` are 0–1; `wan_gateways[]` is `up` \| `down` \| `warn`; `suricata` is `running` \| `stopped` \| `unknown`. Map to judgment as below. Tune thresholds with acceptance tests; do not invent extra health keys.

| Signal | Match (defaults until tuned) | Judgment | Proposal |
|--------|------------------------------|----------|----------|
| Suricata down / critical | `suricata: "stopped"` | `severity: high`, `classes: ["health"]` | `health.restart_service` `{ node, unit: "suricata" }` — **`propose` first** in v0.1 |
| Disk critical | `disk` ≥ **0.95** | `high` / `health` | `notify.operator` `reason_code: health_disk` — **no** auto delete, no disk tools |
| WAN gateway down | any `wan_gateways[]` is `down` | `critical` / `health` | notify only (do **not** flap routes from rules) |
| CPU sustained critical | `cpu` ≥ **0.95** for **≥ 2** consecutive windows | `medium` / `health` | observe / notify; `health.set_nvpmodel` is **out of band** until a Host execute path exists |

`suricata: "unknown"` → notify only (do not propose restart). `wan_gateways` `warn` → `observe` (no ticket) unless a later amendment raises it.

## Policy (conservative)

- **Auto-execute off** for health tools in v0.1. Optional later: Suricata restart `execute` after **N** failed checks + cooldown (explicit amendment).
- Default: `propose` the companion health tool (when one exists) **and required** `notify.operator` → `fyber.auditor` (reuse the notify door / [plane ticket](fyber-auditor-api-v0.md)).
- **Cooldown** per `(node, unit)` against restart storms: a second `health.restart_service` propose for the same pair inside the window is **suppressed** (`observe`; no second ticket).
- Reject unknown tools / bad args (JSON Schema). Do not add `shell.exec`, disk wipes, or route flaps.
- IDS rate-limit **B** does not apply here; health has its own cooldown, not the block-per-hour cap.

If policy would emit an ack-required `propose` **without** a valid `notify.operator`, that is a policy bug: do not pretend the companion was offered; still write a receipt recording the miss.

Notify door (channel `fyber.auditor`, queue-if-offline, human resolution): [notify.operator ↔ shared auditor door](notify-operator-audit-door.md). Ticket `display.tool` is the companion health tool when one is proposed (`health.restart_service`). Auditor v0 `patch` allowlist is **none** for health tools — `amended` cannot change restart args.

## Locked tools

From `schemas/common.v0.json`. Do not extend. Do not expand the v0 catalog.

| Tool | Args | v0.1 mode |
|------|------|-----------|
| `health.restart_service` | required `node`, `unit` | `propose` only |
| `health.set_nvpmodel` | required `device_id`, `profile` | deferred — **do not execute**; Host path does not exist yet |
| `notify.operator` | `channel`, `severity`, `text_redacted` as in the auditor door | `execute` (notify actuator) |

`additionalProperties: false` on args. Extra keys fail schema validation.

### Example ToolCalls (Suricata down)

```json
{
  "call_id": "550e8400-e29b-41d4-a716-446655440071",
  "tool": "health.restart_service",
  "mode": "propose",
  "reason_code": "health_suricata_down",
  "args": {
    "node": "opnsense-cottage",
    "unit": "suricata"
  }
}
```

```json
{
  "call_id": "550e8400-e29b-41d4-a716-446655440072",
  "tool": "notify.operator",
  "mode": "execute",
  "reason_code": "health_suricata_down",
  "args": {
    "channel": "fyber.auditor",
    "severity": "high",
    "text_redacted": "health_suricata_down site=net-tn-cottage node=opnsense-cottage unit=suricata"
  }
}
```

## Receipt (`fyber.receipt/v0`)

| Field | Value |
|-------|--------|
| `door` | `ops` |
| `purpose` | `health` |
| `policy.decision` | `propose` (or `observe` on cooldown / no-op) |
| `policy.engine` | `brewnix-policy/v0` |
| `policy.rule_ids` | detector ids (e.g. `["health_suricata_down"]`) |
| `human.required` | `true` when notify is required |
| `input.sources` | include `opnsense` and/or `suricata` as appropriate; no raw EVE |

Notify `execution[]` follows the auditor door (`executor: "auditor-api@site"`, `effect.queued` when the plane is down). Companion `health.restart_service` is **not** applied in v0.1.

## Acceptance tests

1. Synthetic bundle with `suricata: "stopped"` → propose `health.restart_service` **and** required notify; **zero** execute of the restart unless a later amendment explicitly enables it.
2. Disk critical (`disk` ≥ 0.95) → notify only (`reason_code: health_disk`); no destructive tools (no delete, no `shell.exec`).
3. Cooldown: second restart propose for the same `(node, unit)` inside the window is suppressed (`observe`).
4. JSON Schema validate envelopes / receipts (local `$id` → `schemas/` map). `needs_human: true` and `human.required: true` when notify is required.
5. `plane_reachable: false` → local receipt still appended; notify `effect.queued: true`; queue drains later without rewriting the original receipt.

## Non-goals

- Full Ansible remediation
- PAIR / model triage (`actor.kind: "model"`)
- Quarantine VLAN (`net.quarantine_host` — needs proxmox-firewall VLANs first)
- Expanding the v0 tool catalog
- Auto-execute health tools in v0.1
- Flapping WAN routes or disk deletion from rules

## Suggested implementation layout

```
brewnix-site-defense/
  rules/health-v0.1.yaml
  policy.py                    # health auto-execute off; cooldown map
  sensors/eve_to_bundle.py     # already fills health.*
  exec/notify_auditor.py       # slice 3
  receipt/chain.py
  schemas/                     # pin Brewnix/inference-iface
```

## Change control

Additive clarifications to this doc are fine. Behavioral changes (auto-execute on, new health tools, route flaps, or treating `unknown` Suricata as a restart) need an explicit amendment. Semantic changes to locked schemas remain **v1** or a Chris-approved schema amendment.
