# Model triage v0

**Status:** working spec — **LOCKED 2026-09-07** (Chris; after pressure-test + amendment)  
**Schemas:** pin this repo (`fyber.inference_iface/v0`, `fyber.receipt/v0`, `fyber.feature_bundle/v0`, `fyber.common/v0`) — **no v0 schema break**  
**Site contract:** model-triage policy + engine list (docs-first; **not** a file under `schemas/`)  
**Companion:** [Zero-LLM OPNsense detect → block → receipt loop](zero-llm-opnsense-loop.md) (hot path; still contains with no model) · [fyber.privilege_grant v0](privilege-grant-v0.md) (`allow_model_execute` + `model_tier`) · [fyber.incident binding v0](incident-binding-v0.md) (model hold/propose opens/joins `security`) · [notify.operator ↔ shared auditor door](notify-operator-audit-door.md)

Goal: a **judge-only** local model on the **same** pipeline as the zero-LLM loop. The model may fill `fyber.inference_iface/v0`. Policy + typed actuators still execute. Hot critical auto-contain stays rule-deterministic. Prompts never leave the site box.

**Home:** this repo holds the contract. Site JSON Schemas stay locked; do not add triage fields, `needs_model`, grant tools, or `incident_id` to `schemas/`.

## Goal / non-goals

| Goal | Meaning |
|------|---------|
| Judge only | LLM never sits in the executor path. Envelope = judgment + typed `proposals`. |
| Same pipeline | `feature_bundle` → envelope → policy → executor → receipt. No side door. |
| Rules stay first | Critical auto-contain works with **no model process**. Auto-execute **rule ids** short-circuit regardless of profile. |
| Single winner | One envelope to policy. Never merge rules + model proposals. |
| Local default | ~95% of calls stay `local_*`. `plane_ir` / `host_leased` only under an active grant `model_tier`. |

| Non-goal | Why |
|----------|-----|
| Replacing the zero-LLM path | [Zero-LLM loop](zero-llm-opnsense-loop.md) remains the first executor path. |
| ChatOps agent | No conversational actor. No renter chat. |
| Expanding `ToolName` | Locked enum. Do **not** add a grant tool to `schemas/common.v0.json` yet. |
| Requiring Jetson / PAIR | PAIR is an optional engine under `engines[]`, not an actor, not a tier. |
| `model_primary` as a mode name | CUT. The explicit site mode is `model_assist`. Not the home default. |
| JSON Schema under `schemas/` | Implement against this file. |

## Axioms

1. **LLM judges only — never in the executor path.** Policy + typed actuators apply. The model is never `resolved_by` on a grant and never opens an [incident](incident-binding-v0.md) record (`opened_by.kind` stays `rule` \| `automation` \| `human`).
2. **Same pipeline.** `feature_bundle` → envelope → policy → executor → receipt. A later engine fills the same `fyber.inference_iface/v0` the rules pack already emits.
3. **Hot critical auto-contain is rule-deterministic.** It works with no model process present. Auto-execute **rule ids** (`port_scan_burst`, …) short-circuit **before** any engine call.
4. **Prompts are ephemeral and site-local.** Never in receipts, tickets, grants, incidents, or the plane. Same redaction doctrine as the locked schemas.
5. **~95% local.** Default tier is `local_small`. `plane_ir` / `host_leased` only when an **active** [privilege grant](privilege-grant-v0.md) minted `model_tier` to that value.
6. **PAIR is an optional engine, not an actor.** It appears under `engines[]` (`kind: pair`). It is not `actor.kind`, not a `model_tier`, and not a required box.

## `triage_mode`

Site config enum. **Not** a schema field.

| Mode | When | Behavior |
|------|------|----------|
| `rules_only` | Zero-LLM. No engine configured, or site pinned this way. | Never call a model. [Zero-LLM loop](zero-llm-opnsense-loop.md) only. |
| `rules_primary` | **v0 default** when at least one engine is configured. | Rules run first. Call the model only on the gates below. Auto-execute still short-circuits. |
| `model_assist` | Explicit site config only. **Not** the home default. **Not** named `model_primary`. | Model fills **non-auto** gaps. Rules still short-circuit on auto-execute. |

`model_primary` is not a v0 token. Treat it as a typo → reject config.

## When to call the model (`rules_primary`)

Call the model when **any** of the following hold:

1. **No auto-execute rule matched** **and** `severity` ≥ `high` **and** the match is **not** whitelist / `noise_ignore`.
2. **Optional enrich** (default **off**): a propose-only / `hold_human` rule matched **and** site `triage.enrich` is `true`.

**NEVER** call the model when a **critical auto-execute** rule already matched.

**CUT:** a free `needs_model` sensor flag on the bundle (or anywhere else). The call decision is derived from rules + severity + `triage.enrich`. Do not add the flag to `schemas/feature_bundle.v0.json`.

`model_assist` uses the same **never-call-on-auto-execute** gate. It may call on other non-auto gaps even when severity is below `high` — only if the site config that enabled `model_assist` says so. Home sites should not turn this on.

## Single winner (never merge proposals)

Exactly one envelope is handed to policy. Two brains are never fused.

| Situation | Envelope to policy |
|-----------|-------------------|
| Auto-execute rule hit | **Rules** only. Model is not called. |
| `rules_primary`, model not called | **Rules** |
| `rules_primary`, model schema-OK | **Model** (rules stay on a site-local **eval-log** only — not in the envelope, not merged into `proposals`) |
| `rules_primary`, model fail | **Rules** if that envelope is usable; else [fallback](#fallback) |
| `model_assist`, model OK | **Model** for non-auto; **rules** for auto (auto already short-circuited) |
| `model_assist`, model fail | **Rules** if usable; else fallback |

Eval-log is an implementation file beside the receipt store (e.g. `/var/lib/brewnix/triage_eval.jsonl`). It is **not** a schema object. Do not stuff prompts into it. Optional later: `receipt.annotate` with a redacted pointer — not a v0 requirement.

## `allow_model_execute` (Chris amendment)

Derived by policy from the active [grant](privilege-grant-v0.md) `rails_profile`. **Not** an envelope field.

| Grant / profile | `allow_model_execute` | Model-originated companion tools |
|-----------------|----------------------|----------------------------------|
| `rails_profile: strict` **or no grant** | **`false`** | Model may only produce **`propose` / `hold_human` / notify** paths. Policy **must not** auto-execute model-originated companion tool calls (`firewall.block_ip`, `health.restart_service`, …). |
| **Active** grant (`approved` and now < `active_until`) with minted `rails_profile` **`ir_elevated` or `break_glass`** | **`true`** | Execute-eligible only for tools in the **active allowlist / profile max catalog**. Still subject to whitelist, rate limit, confidence θ, subject-bind, and JSON Schema. |

`notify.operator` on a hold / ack-required propose is the [auditor door](notify-operator-audit-door.md), not “model execute.” It stays required on those hooks even when `allow_model_execute` is `false`.

Further locks:

- Auto-execute **rule ids** remain **rules-only** short-circuit **regardless of profile**. Elevation does not let the model steal that path.
- **Model-originated `critical` ≠ auto-rule `critical`.** A model saying `severity: critical` does not enter the auto-execute set. It still needs `allow_model_execute`, an allowlisted tool, θ, and the other gates.
- Elevated execute after `active_until` is **denied** (grant spec). Policy returns to `strict` → `allow_model_execute: false`.

Do not add a grant tool to `common.v0` in this lock. Cite a grant with `receipt.annotate` (or the [incident](incident-binding-v0.md) side index) as in the grant spec.

## Fallback

On model failure, schema-invalid output, or `engines[]` exhausted (`on_model_failure`):

```
if auto_rule_matched:          use rules
elif rules_envelope_usable:    use rules
elif severity >= high:         hold_human + notify.operator (channel fyber.auditor)
else:                          observe
```

- **≤ 1** schema-repair retry, then fallback. No second repair.
- PAIR **preempt** (or any PAIR-down) → `engine_unavailable` → next engine in `engines[]`, else this fallback.

**FOREVER CUT** (not aliases, not flags, not “debug only”):

| Token | Why |
|-------|-----|
| `fail_open_execute` | Never execute because the model or engine failed. |
| Unbounded retry | One repair then fallback. |
| `execute_last_rules` | Do not execute a stale rules envelope that was already superseded or marked unusable. “Rules if usable” is the only rules recovery. |

## Confidence

- Default **θ = 0.6** for **execute** eligibility, and only when `allow_model_execute` is `true`.
- Below θ → `propose` / `hold_human` (plus required notify on the ack/hold hooks).
- Rules envelopes keep their existing confidence (`1.0` or fixed `0.9`).
- **Unknown tools are stripped** before policy (not forwarded, not executed). Remaining proposals must still pass `inference_iface.v0`. An empty `proposals` after strip is valid; policy then `observe` or hold per the fallback / notify hooks.

## Engines (config sketch)

Docs-first site config. Extra unknown engine kinds fail validation. PAIR is not a `model_tier`.

```yaml
triage:
  triage_mode: rules_primary    # v0 default when an engine is configured
  default_tier: local_small     # local_small | local_large; plane_ir / host_leased need a grant
  enrich: false                 # optional propose/hold enrich; default off
  confidence_theta: 0.6
  engines:                      # ollama | pair | llama.cpp  (ordered)
    - kind: ollama
      id: qwen2.5-7b-q4
      tier: local_small
    - kind: pair
      id: pair-local
      tier: local_large
  on_model_failure: fallback    # the algorithm above — never fail_open_execute
```

`default_tier` is the resting tier. A minted grant `model_tier` may raise it for that `(site_id, incident_id)` until `active_until`. `budget_tokens` on the same grant is the token cap (grant spec). `prompt_route` selects **site-local** template ids; the template bodies stay ephemeral and never enter the receipt.

## I/O

**In** (to the model process; not persisted on the envelope):

| Input | Constraint |
|-------|------------|
| Feature bundle | Redacted `fyber.feature_bundle/v0` only. No EVE payloads, packet captures, or prompts. |
| `incident_id` | Optional. When present, the open [incident](incident-binding-v0.md) this cycle may join. Not an envelope field in v0. |
| `rails_profile` | `strict` \| `ir_elevated` \| `break_glass` (resting or minted). Drives `allow_model_execute`. |
| Allowlisted tools | Locked `ToolName` ∩ (site baseline ∪ active grant catalog). |

**Out:**

- Validate `fyber.inference_iface/v0` (`schemas/inference_iface.v0.json` + `common.v0.json`; local `$id` map).
- `actor.kind` **must** be `model`. `actor.id` is a stable engine id (e.g. `qwen2.5-7b-q4@sha256:…`). `purpose` is `triage` in v0.
- `inputs_digest` = `features_digest` of the cycle’s bundle (canonical JSON; `sha256:` + 64 hex).
- Every `judgment.subjects[].value` and IP-bearing tool arg **must** appear in the bundle (`top_subjects` / `counts.by_src` / declared health subjects). Enforced by policy, not only JSON Schema.
- Schema-invalid → repair once or fallback. Do not forward a failing envelope.

Shape reference: `examples/model-triage.envelope.example.json` (strict / `allow_model_execute: false` — propose + notify). `examples/envelope.example.json` remains a generic `actor.kind: model` shape; it is **not** a license to execute under `strict`.

## Incident / grant

- Model `hold_human` / ack-required `propose` **opens or joins** a `security` incident per [incident-binding-v0](incident-binding-v0.md) (same trigger as rules hold/propose + required notify). Policy / automation writes the record. The LLM does not.
- Quiet observe / whitelist / `noise_ignore` still **do not** open an incident.
- Elevated model **execute** only while an **active** grant has minted `ir_elevated` or `break_glass` (`allow_model_execute: true`) and the tool is in the active catalog.
- Grant propose / mint still requires `incident_id`. Do **not** add a grant tool to `common.v0` in this lock.

## Eval harness

Offline. Replay recorded bundles. No live plane, no live OPNsense required.

| Check | Pass |
|-------|------|
| Schema rate | Every model envelope validates `inference_iface.v0` (or was dropped into fallback after ≤1 repair). |
| Subject-bind | Zero IP / subject leaks outside the bundle. Violations fail the run. |
| Dry-run over-execute | Under `strict`, **zero** model-originated companion `execute`. Auto-rule executes remain rules-only. |
| Offline | Same fixtures with no engine and with `plane_reachable: false` still contain on critical rules and still write receipts. |

Prompts used in a replay stay on the eval host. They are not checked into this repo and are not copied onto receipts.

## Acceptance tests

1. **No model process → critical rules still block.** Stop every engine. Synthetic `port_scan_burst` → `execute` → alias add + receipt. Same as zero-LLM test 4.
2. **Auto-execute match → model not called.** Instrument the engine. A critical auto rule must produce **zero** model invocations.
3. **Model schema fail → fallback; no execute from the model.** After ≤1 repair, follow [fallback](#fallback). No companion `execute` attributed to the model.
4. **`strict` (or no grant) → model propose only.** `allow_model_execute: false`. Model-originated `firewall.block_ip` / health tools stay `propose` or hold; notify door may still `execute`.
5. **Active `ir_elevated` / `break_glass` → model execute only for allowlisted tools + policy gates.** θ, whitelist, rate limit, subject-bind, schema all still apply. Tool outside the catalog → stripped / rejected. After `active_until`, back to test 4.
6. **PAIR down → next engine or fallback; no fail-open.** Preempt = `engine_unavailable`. Never `fail_open_execute`.
7. **Two brains never merged.** Policy sees either a rules envelope or a model envelope, never a concatenated `proposals` array from both.
8. **Receipt has no prompt.** Receipt / ticket / grant / incident bodies contain no prompt text, EVE, or packet payload.

## Change control

Additive clarifications to this doc are fine. A `model_primary` mode, `needs_model` on the bundle, `fail_open_execute`, merging rules+model into one envelope, a grant tool on `common.v0`, or putting prompts on the receipt need an explicit Chris amendment. Semantic changes to locked files under `schemas/` remain **v1** or a Chris-approved schema amendment.

Implement against this file. Do not wait on a schema file under `schemas/`.
