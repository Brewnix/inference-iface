# Hypermesh preempt v0

**Status:** working spec — **LOCKED 2026-09-07** (Chris; after pressure-test + H3 drain-executor amendment + H4 owner-actuator inventory). Additive CP cite **2026-09-08**: Panopticon phase-2 [#40](https://github.com/FyberLabs/panopticon/pull/40) / [#41](https://github.com/FyberLabs/panopticon/pull/41) on **main** (descriptive Host/CP path only; **does not** weaken locked gates).  
**Schemas:** pin this repo (`fyber.inference_iface/v0`, `fyber.receipt/v0`, `fyber.feature_bundle/v0`, `fyber.common/v0`) — **no v0 schema break**; tools already in `common.v0`  
**Site / Host contract:** Hypermesh preempt policy + Host job path (docs-first; **not** a new file under `schemas/`)  
**Companion:** [fyber.privilege_grant v0](privilege-grant-v0.md) (explicit `tool_allowlist_add`; **never** implied by `rails_profile`) · [Model triage v0](model-triage-v0.md) (LLM judges only; PAIR is an engine) · [fyber.incident binding v0](incident-binding-v0.md) (`lease_stop` execute needs an open `incident_id`) · [Zero-LLM OPNsense detect → block → receipt loop](zero-llm-opnsense-loop.md) (contain stays OPNsense; this spec is not that loop) · [SociACL IR binding v0](sociacl-ir-binding-v0.md) (`delegate execute` on `:host` when Host exists; Checkout **not** a SociACL consumer) · [Panopticon sell-pause-v0](https://github.com/FyberLabs/panopticon/blob/main/products/hypermesh/docs/sell-pause-v0.md) (CP job + result schema) · [Panopticon brewnix-executor-bridge-v0](https://github.com/FyberLabs/panopticon/blob/main/products/hypermesh/docs/brewnix-executor-bridge-v0.md) (site-token enqueue+poll; phase-2 [#40](https://github.com/FyberLabs/panopticon/pull/40) on **main**) · [Panopticon heartbeat-sell-state-v0](https://github.com/FyberLabs/panopticon/blob/main/products/hypermesh/docs/heartbeat-sell-state-v0.md) (site-token live `sell_state` read; phase-2 [#41](https://github.com/FyberLabs/panopticon/pull/41) on **main**)

Goal: a **typed preempt path** for Hypermesh sell/lease actuators. Brewnix **proposes**. The Host **executes**. Policy decides `propose` vs `execute` with **asymmetric** gates: `sell_pause` blast is smaller than `lease_stop`. Drain sequencing is **Brewnix H3** — two CP jobs, not a Host mega-job. The LLM never executes. PAIR is not this path.

**Home:** this repo holds the **Brewnix → Host API** contract (MIT / OSS intent). **`FyberLabs/hypermesh-host` implements.** H1 (`lease_stop` identity), H2 (`sell_pause`), and H4 (offline owner actuators) are on Host **main**. CP `sell-pause-v0` ([#37](https://github.com/FyberLabs/panopticon/pull/37)), site-token executor bridge ([#40](https://github.com/FyberLabs/panopticon/pull/40)), and site-token heartbeat `sell_state` read ([#41](https://github.com/FyberLabs/panopticon/pull/41)) are on Panopticon **main**. See [Host reality (2026-09-07)](#host-reality-2026-09-07). Site JSON Schemas stay locked; do **not** change `schemas/`.

## Goal / non-goals

| Goal | Meaning |
|------|---------|
| Typed proposals | `hypermesh.lease_stop` / `hypermesh.sell_pause` on `fyber.inference_iface/v0` using locked `common.v0` args. |
| Asymmetric gates | `sell_pause` may auto-execute under narrow strict+rules conditions. `lease_stop` never auto-executes on `strict`. |
| Explicit allowlist | Hypermesh tools are **never** implied by `rails_profile` alone. Need `tool_allowlist_add` or a human. |
| Offline dignity | Same envelope / receipt JSON with the plane down. Owner console must work plane-down **as policy**. |
| Host executes | Brewnix does not call OPNsense for these tools. Host (via the job path below) applies the effect. |

| Non-goal | Why |
|----------|-----|
| Changing `schemas/` | Tools already exist in `common.v0`. This lock is policy + Host path. |
| Market discovery / payments | Different plane. README non-goal unchanged. |
| SociACL Check grants | Different plane. Binding: [sociacl-ir-binding-v0](sociacl-ir-binding-v0.md). Checkout is **not** a SociACL consumer of these grants. Owner-console ack (when Host exists) is `delegate execute` on `:host`. |
| Implying tools via profile | Axiom 6. Profile is a ceiling, not a Hypermesh grant. |
| Mass-stop without incident / policy | Rate limit + incident gate + strip extras. |
| Treating iface args as today's Host RPC | Host owner POSTs reuse H1/H2 actuators. They are **not** `{lease_id, reason_code}` and **not** a tool named `hypermesh.sell_pause`. See [Host reality](#host-reality-2026-09-07). |

## Axioms

1. **Owner > renter > market.** Owner console / site-owner capability outranks renter leases and market sell.
2. **Brewnix proposes; Host executes.** Not OPNsense. These tools are not the zero-LLM contain loop.
3. **Plane optional for act; offline dignity.** Policy and receipts must work with `plane_reachable: false`. Host **H4** now satisfies the owner-console path. The lock does not drop the axiom if the plane is down.
4. **LLM judges only; not PAIR.** Model fills `fyber.inference_iface/v0`. PAIR is an optional engine ([model-triage](model-triage-v0.md)), not this actuator, not `actor.kind`.
5. **`sell_pause` blast < `lease_stop` blast.** Asymmetric execute gates. Drain (`sell_pause` first) before stop.
6. **Hypermesh tools are NEVER implied by `rails_profile` alone.** Execute eligibility needs an explicit [`tool_allowlist_add`](privilege-grant-v0.md) **or** a human (auditor ticket / owner console). `ir_elevated` default max catalog does **not** include `hypermesh.*`.

## Host reality (2026-09-07)

Inventory from **Developer Bot (Host)** plus what landed on **main** the same day (H1, H2, **H4**). Panopticon phase-2 **#40** (site jobs) and **#41** (site device `sell_state` read) later landed on Panopticon **main**. This section is **descriptive of today's Host / CP**. It does **not** weaken the locked Brewnix policy gates above. It clarifies the **execution path**. Remaining Host gaps: **no** `preempt_mode=hard` signal; Path B is **not** sell (**AGX Path B remains the live join gate**).

Brewnix iface tools are the **proposal contract**. They are **not** names of local Host binaries or RPCs.

| Fact | Today (Host / CP) | Do not assume |
|------|-------------------|---------------|
| `lease_stop` | Job kind `lease_stop` on Host **main** ([PR #5](https://github.com/FyberLabs/hypermesh-host/pull/5) — H1 identity). **Not** a local tool named `hypermesh.lease_stop`. Host **polls** `GET` jobs, **matches `lease_id` to the local handle**, then `docker stop` on `hypermesh-<lease_id>`, `POST` `{passed, image_hash?}`. Handle present + mismatch / empty id → `passed=false`, no stop. No handle → soft no-op `passed=true`. **No `reason_code` on job or result.** H4 owner `POST /v0/lease_stop` reuses this actuator. | Host accepts `{lease_id, reason_code}` as a local call. |
| `sell_pause` | Job kind `sell_pause` on Host **main** ([PR #6](https://github.com/FyberLabs/hypermesh-host/pull/6) — H2). CP schema + enqueue on Panopticon **main** ([PR #37](https://github.com/FyberLabs/panopticon/pull/37), [`sell-pause-v0`](https://github.com/FyberLabs/panopticon/blob/main/products/hypermesh/docs/sell-pause-v0.md)). Same agent-jobs channel as `lease_stop`. Host persists `$STATE_DIR/sell.json`; matching `device_id` → `paused` + optional `until`; heartbeat reports `sell_state`. Result `{passed, sell_state, until?, device_id?}`. `passed=true` + `n/a` is invalid. H4 owner `POST /v0/sell_pause` reuses this actuator. | A local RPC named `hypermesh.sell_pause`, or that Path B pauses selling. |
| Path B | Job kind `path_b` / CP `path_b_state`. **Not** sell posture. **AGX Path B remains the live join gate.** | `path_b` / `posture.path_b` is `sell_state`, or that a Path B job pauses selling. |
| Offline / plane-down | **H4** on Host **main** ([PR #7](https://github.com/FyberLabs/hypermesh-host/pull/7)). Unix socket `$STATE_DIR/owner.sock` + bearer `$STATE_DIR/owner.token` **0600**. `POST /v0/sell_pause` and `POST /v0/lease_stop` reuse H1/H2 (not CP enqueue). `$STATE_DIR/owner-effects.jsonl` for plane-down effect lines. CLI `hypermesh-agent owner …`. Do **not** require the site-token `GET` / `/site/jobs` plane-up doors when `plane_reachable: false`. | Owner path is CP job enqueue, loopback TCP, or that Host writes `fyber.receipt`. |
| Site-token plane-up (Brewnix) | Panopticon **[#40](https://github.com/FyberLabs/panopticon/pull/40)** ([`brewnix-executor-bridge-v0`](https://github.com/FyberLabs/panopticon/blob/main/products/hypermesh/docs/brewnix-executor-bridge-v0.md)): `POST`/`GET /api/v1/hypermesh/site/jobs` with site `hm_site_…`. Existing kinds `lease_stop` / `sell_pause` only. Site-bridge `lease_stop` **requires** `lease_id` (422 if missing/empty; no omit-for-active) — same as the iface tool. `Device.site_id` bind scopes enqueue. Panopticon **[#41](https://github.com/FyberLabs/panopticon/pull/41)** ([`heartbeat-sell-state-v0`](https://github.com/FyberLabs/panopticon/blob/main/products/hypermesh/docs/heartbeat-sell-state-v0.md)): `GET /api/v1/hypermesh/site/devices/{device_id}` returns `{device_id, site_id, sell_state, last_heartbeat_at}`. Persist already existed; this is the Brewnix read. **Not** a `fyber.receipt` write. CP echoes last known + `last_heartbeat_at`; **Brewnix** decides staleness (docs-side only — **not** a schema field). | Brewnix plane-up execute uses the operator-only `POST /jobs` door, that enqueue is apply, or that this GET writes receipts. |
| Hard preempt | Host has **no** `preempt_mode=hard` signal. v0 never selects `hard`. | Host emits a hard-preempt signal. |
| Drain / identity / receipts | **H1** lease identity is on Host. **H2** `sell_pause` reports `sell_state` on the job result and heartbeat. **H4** appends `owner-effects.jsonl` (not `fyber.receipt`). **No** Host drain mega-job — drain is **Brewnix [H3](#h3-drain-executor-locked-2026-09-07)** (two CP jobs; executor sequences via the site-token `/site/jobs` door; last-known `sell_state` via the site GET). Soft-idempotent if **no handle**. `lease_stop` result stays `{passed, image_hash?}` — **no** `sell_state` / `path_b` on that result. | Host orders drain then stop as one job, or that `lease_stop` returns posture fields. |

### v0 Host path (what actually runs)

1. **Proposal contract (this repo):** envelope `ToolCall`s `hypermesh.lease_stop` / `hypermesh.sell_pause` with locked args. Policy applies the [execute vs propose matrix](#execute-vs-propose-matrix) unchanged.
2. **`lease_stop` execute (today, plane-up):** Brewnix presents site `hm_site_…` and **`POST /api/v1/hypermesh/site/jobs`** `{kind: lease_stop, device_id, lease_id}` ([#40](https://github.com/FyberLabs/panopticon/pull/40), [`brewnix-executor-bridge-v0`](https://github.com/FyberLabs/panopticon/blob/main/products/hypermesh/docs/brewnix-executor-bridge-v0.md)). `lease_id` is **required** (422 if missing/empty; no omit-for-active) — same as the iface tool. Poll `GET /api/v1/hypermesh/site/jobs/{job_id}`. Host H1-matches `lease_id` to the local handle, then `docker stop`s `hypermesh-<lease_id>`, posts `{passed, image_hash?}`. Brewnix / policy maps that result onto `execution[]`. Do **not** send `reason_code` as a Host job field; keep it on the envelope / receipt. Do **not** treat the operator `POST /jobs` door as the Brewnix path.
3. **`sell_pause` execute (today, plane-up):** same site-token door `{kind: sell_pause, device_id, until?}`. Host H2 applies pause for matching `device_id` and posts `{passed, sell_state, until?, device_id?}`. Heartbeat reports current `sell_state`. Do **not** reuse job kind `path_b`. Do **not** invent `paused` from enqueue alone.
4. **Drain (H3):** **Brewnix sequences** the two jobs (not CP). Not a new Host job kind. Plane-up last-known `sell_state` is `GET /api/v1/hypermesh/site/devices/{device_id}` ([#41](https://github.com/FyberLabs/panopticon/pull/41)). See [H3 Drain executor](#h3-drain-executor-locked-2026-09-07).
5. **Offline owner actuators (H4):** unix socket `$STATE_DIR/owner.sock` + bearer `$STATE_DIR/owner.token` **0600**. Owner console / `hypermesh-agent owner …` POSTs `/v0/sell_pause` and `/v0/lease_stop` on this box. Same H1/H2 actuators; **no** CP enqueue. Append `$STATE_DIR/owner-effects.jsonl` (not `fyber.receipt`). Site still writes `fyber.receipt/v0` from those lines. Plane-up execute still uses Panopticon site jobs. Do **not** require the site GET / `/site/jobs` plane-up doors when `plane_reachable: false`. Axiom 3 stays: Brewnix policy works plane-down; Host now satisfies the owner-console path.

## Tools (`common.v0` — do not extend)

From `schemas/common.v0.json`. Extra keys fail schema validation. Do not add fields to `schemas/`.

| Tool | Required args | Optional | `reason_code` |
|------|---------------|----------|----------------|
| `hypermesh.lease_stop` | `lease_id`, `args.reason_code` | — | ToolCall `reason_code` **and** `args.reason_code` (schema requires both). Policy allow-set below. Host job has **neither**. |
| `hypermesh.sell_pause` | `device_id` | `until` (RFC3339) | ToolCall `reason_code` only (not in args). |

`mode` is `dry_run` \| `propose` \| `execute`. Policy may downgrade `execute` → `propose` / `hold_human`. Unknown `reason_code` → strip / reject that call (do not execute).

Shape reference: `examples/hypermesh-preempt.example.json`.

### `reason_code` allow set

Exactly:

`site_defense` \| `owner_primary` \| `health_evacuate` \| `owner_stop_selling` \| `incident_preempt`

Unknown token → not execute-eligible. Schema still allows any `^[a-z][a-z0-9_]{0,63}$`; **policy** enforces this set.

## Execute vs propose matrix

`rails_profile` never grants these tools by itself (axiom 6). Elevated rows still require the tool on an active [`tool_allowlist_add`](privilege-grant-v0.md) (or a human). `hypermesh.*` is **not** in the `ir_elevated` default max catalog.

### `hypermesh.sell_pause`

| Condition | Decision |
|-----------|----------|
| `strict` + **rules** + `reason_code` ∈ `{health_evacuate, owner_stop_selling}` + site flag `allow_sell_pause_execute` | **May `execute`**. Flag **defaults true for health** (`health_evacuate`). |
| `strict` + **model** | **Propose only** (`allow_model_execute` is false on strict / no grant). |
| Elevated (`ir_elevated` \| `break_glass`) + tool **allowlisted** + (rules **or** model with `allow_model_execute`) | **May `execute`** (still schema, allow-set, rate / strip limits). |
| Human (auditor ticket approve **or** owner console ack) | **Execute** (policy + Host path). |
| Security incident | **Not required** for `sell_pause`. **Ops** incident preferred for health ([binding](incident-binding-v0.md)). |

H2: when policy says `execute`, Panopticon enqueues `kind=sell_pause`; Host applies and reports `sell_state`. Do not invent a Path B job. Drain vs stop is [H3](#h3-drain-executor-locked-2026-09-07). See [Host reality](#host-reality-2026-09-07).

### `hypermesh.lease_stop`

| Condition | Decision |
|-----------|----------|
| `strict` (rules **or** model) | **Propose + notify only.** Execute **only** via human auditor ticket approve **or** owner console ack. |
| Elevated + tool **allowlisted** + `reason_code` ∈ allow set + **open `incident_id` required** + rate limit OK + (rules **or** model + `allow_model_execute`) | **May `execute`**. v0 Host path = Panopticon `kind=lease_stop` job. |
| `break_glass` | Same gates as elevated. **Enterprise dual-control.** Always keep an audit ticket trail. |
| No open incident | **Force propose** even under `break_glass`. |

`notify.operator` on propose / hold is the [auditor door](notify-operator-audit-door.md). Required on those hooks.

## H3 Drain executor (LOCKED 2026-09-07)

**Chris amendment** after pressure-test. Drain is **not** a Host mega-job. Host keeps **two** CP job kinds (`sell_pause`, `lease_stop`). **Brewnix sequences.** Do not add a third Host job kind.

`preempt_mode` (docs-first site config; **not** a schema field): `drain` \| `hard`.

- **`drain`** — default. Pause sell, then stop leases. This section.
- **`hard`** — only for `reason_code: site_defense` **and** only when **Host says needed**. Host has **no** such signal — **v0 never selects `hard`**. Do not self-promote. Do not invent a Host hard signal.

Site flag `allow_stop_while_selling` (docs-first; **not** a schema field) **defaults false**. The `site_defense` pause-fail exception below treats that path as true **only** for that path.

### Algorithm

Filter **execute-eligible** `hypermesh.*` proposals (matrix, allowlist, incident, rate / strip already applied).

1. **Sort:** all `sell_pause` before any `lease_stop`. Stable within each group. **Device-scoped.**
2. **Per device:** await **each** `sell_pause` job **result** before enqueueing **any** `lease_stop` for that device.
3. **Different devices** may run in parallel.
4. **Prefer sync await** of the pause result in the same cycle. **Deferred** (pause this cycle; enqueue stops on a later cycle after last-known `sell_state` is `paused`) is a documented **fallback only** — not the default. Plane-up, that last-known is `GET /api/v1/hypermesh/site/devices/{device_id}` ([heartbeat-sell-state-v0](https://github.com/FyberLabs/panopticon/blob/main/products/hypermesh/docs/heartbeat-sell-state-v0.md)): Host heartbeat or `sell_pause` result write. Copy the Host-reported value; do **not** invent `paused` from enqueue alone.

Policy / executor must not apply stop before pause on that device in that cycle.

### Pause success

Treat pause as success only when the Host result is `passed=true` **and** `sell_state` ∈ `{paused, draining, off}`.

- `selling` is **not** success.
- `n/a` with `passed=true` is **invalid** ([sell-pause-v0](https://github.com/FyberLabs/panopticon/blob/main/products/hypermesh/docs/sell-pause-v0.md)). Do not treat it as paused.

On success, enqueue / run the execute-eligible `lease_stop`s for that device.

### Pause fail / timeout / `passed` but still `selling`

**Default (fail-closed):** no automated `lease_stop`s for that device. `hold_human` + required notify. Receipt records the pause failure (do not invent `paused`).

**Chris amendment — `site_defense` exception:** if `reason_code` is `site_defense` **and** the actor is the **automated rules** path **and** the stops are execute-eligible → **allow stops anyway**. Treat as `allow_stop_while_selling=true` **for this path only**. Still record the pause failure on the receipt. Do **not** auto-select `preempt_mode=hard`. Do **not** invent a Host hard signal.

Other reasons (`owner_primary`, `health_evacuate`, `owner_stop_selling`, `incident_preempt`, …): keep fail-closed (no stops) + `hold_human`.

### Pause propose-only + stop execute

If pause is `propose` (not execute) in this cycle: **do not execute stops** until the pause is human-approved. Same cycle — a held pause does not unlock stop execute.

### Stop-only envelope (no `sell_pause` proposal)

Allow automated `lease_stop` only when last heartbeat / last-known `sell_state` ∈ `{paused, off, n/a}` **or** a human-approved ticket / owner-console ack.

Plane-up, that last-known is `GET /api/v1/hypermesh/site/devices/{device_id}` ([#41](https://github.com/FyberLabs/panopticon/pull/41)): `{device_id, site_id, sell_state, last_heartbeat_at}`. Copy the Host-reported `sell_state` onto receipt posture. Do **not** invent `paused` from enqueue alone. After `until` expiry, Host returns to `selling` on the **next heartbeat**; the old `sell_pause` result stays `paused` — use the GET, not that stale result. CP echoes last known + `last_heartbeat_at`; **Brewnix** decides staleness policy (docs-side note only — **not** a schema field). Plane-down: do **not** require this GET; H4 / last local known still apply.

If `sell_state==selling` **and** the actor is automated:

- **`hold_human`** (+ optional inject a `sell_pause` **propose**) unless:
  - site flag `allow_stop_while_selling` (**default false**), **or**
  - this is the **`site_defense` pause-fail exception** above — **only** when a pause **was attempted and failed** this cycle.

**Clarify:** stop-only while `selling` with a **never-attempted** pause is **not** the `site_defense` pause-fail exception. Still hold unless the site flag or a human.

### Pause OK, stop fails

Keep sell **paused**. **No auto-resume** (Host has no `sell_resume` job in v0). Continue other `lease_stop`s in the batch (per-stop independence). Notify on **any** stop failure.

### `preempt_mode: hard`

Still only `site_defense` **+ Host signal**. Host has no signal. **v0 never selects `hard`.**

## Limits

| Limit | Behavior |
|-------|----------|
| Max **1** `sell_pause` + **≤ 8** `lease_stop` per envelope | Strip extras (keep first valid of each; drop the rest). Do not fail the whole envelope solely for extras. |
| Rate limit `lease_stop`s / hour / site | Suggest **10**. Trip → `hold_human` + required notify. Not the IDS `blocks_per_hour` meter. |

## Owner console

Local Host / Brewnix UI or CLI with **site owner** capability. Resolution on the receipt: `human.resolution` **`approved`**. Must work **plane-down** (axiom 3).

**Host reality:** H4 on Host **main** satisfies this path. Talk to `$STATE_DIR/owner.sock` with `$STATE_DIR/owner.token` (**0600**). CLI: `hypermesh-agent owner sell-pause [--until=…]` / `hypermesh-agent owner lease-stop --lease-id=…`. Effects in `owner-effects.jsonl`. Site maps those onto `fyber.receipt/v0`. Do **not** enqueue a CP job plane-down. Do **not** require the site GET / `/site/jobs` plane-up doors. Do **not** invent `applied` without an owner-effects line or job result.

## Receipts

Same `fyber.receipt/v0` chain as every other door. Idempotent **no-ops are OK** (already paused / already stopped / no handle).

| Field | Preempt notes |
|-------|----------------|
| `door` | `ops` for health / owner-stop-selling; `site_defense` or `ir` when `site_defense` / `incident_preempt` |
| `purpose` | `health` or `triage` (do not invent a new purpose) |
| `posture.sell_state` / `path_b` | Update **from Host effect**. H2 `sell_pause` result and heartbeat carry `sell_state` — copy that. Plane-up live last-known is `GET /api/v1/hypermesh/site/devices/{device_id}` ([#41](https://github.com/FyberLabs/panopticon/pull/41)) — copy Host-reported `sell_state` onto receipt posture. After `until` expiry, use that GET (next heartbeat may be `selling`); do not keep `paused` from the old job result. `lease_stop` result is still `{passed, image_hash?}` — **no** `sell_state` / `path_b` on that result. Do **not** copy CP `path_b_state` into `sell_state`. Do **not** invent `paused` because a Path B job ran or because CP enqueued `sell_pause`. Unchanged / `n/a` / last known is valid when Host has not reported. Stale heartbeat: CP echoes last known + `last_heartbeat_at`; **Brewnix** decides staleness (not a schema field). |
| `input.sources` | include `hypermesh_host` when the cycle is preempt |
| `execution[]` | `executor` like `hypermesh-host@site`, `hypermesh-host@site:owner`, `panopticon:job:sell_pause`, or `panopticon:job:lease_stop`. Plane-down: copy from `$STATE_DIR/owner-effects.jsonl` (Host effect log — not `fyber.receipt`). `effect` may record `device_id` / `lease_id`, `passed`, `sell_state` (pause), optional `image_hash` (stop). Soft no-handle → applied / no-op, not a schema-invalid effect. Pause fail still goes on the receipt even when the `site_defense` exception allows stops. |
| `human` | `required: true` on propose / hold. Owner console / auditor approve sets `resolution: approved`. |

## Privilege-grant composition

See [privilege-grant-v0](privilege-grant-v0.md) profile tables (**amended** by this lock):

- `hypermesh.lease_stop` and `hypermesh.sell_pause` are **not** in the `ir_elevated` default max catalog (`{health.restart_service, notify.operator}` + optional `ids.suricata_pass`).
- They are **not** implied by `ir_elevated` or `break_glass`. Add only via explicit `tool_allowlist_add` when the active profile max catalog includes them (today: only if listed in `packs/emergency-v0`, which **may ship empty**).
- Asking `tool_allowlist_add` of `hypermesh.*` on `ir_elevated` when not in catalog → reject the propose **or** drop that ask on approve.

## Incident binding

See [incident-binding-v0](incident-binding-v0.md). Overlay only — never a contain gate for OPNsense.

| Tool | Incident |
|------|----------|
| `lease_stop` execute | **Open `incident_id` required.** Missing / closed → force propose, including `break_glass`. Usually `security` for `site_defense` / `incident_preempt`; `ops` is allowed when the cycle is health/owner. |
| `sell_pause` | Security incident **not** required. **Ops** preferred for `health_evacuate`. Quiet observe still does not open a case. |

The LLM does not open the record (`opened_by.kind` stays `rule` \| `automation` \| `human`).

## Model triage / PAIR

[Model triage v0](model-triage-v0.md) is unchanged:

- Model may **propose** Hypermesh tools. Under `strict`, policy must not auto-execute them from the model (`sell_pause` execute on strict is **rules-only** + flag + allowed reasons).
- Elevated model execute still needs `allow_model_execute`, an explicit allowlist entry, θ, and the matrix above.
- **PAIR is unaffected.** PAIR preempt / `engine_unavailable` is the triage engine list, not `hypermesh.*`. Do not route Host jobs through PAIR.

## Acceptance tests

1. **Schema-valid both tools.** Envelopes with `hypermesh.lease_stop` (`lease_id` + `args.reason_code`) and `hypermesh.sell_pause` (`device_id`, optional `until`) validate `inference_iface.v0` + `common.v0` (local `$id` map). Extra args fail.
2. **`strict`: `lease_stop` never auto-execute from rules or model.** Decision is `propose` / `hold_human` + notify. Execute only after auditor approve or owner console ack.
3. **`strict`: `sell_pause` execute only for allowed reasons + flag.** `health_evacuate` / `owner_stop_selling` + `allow_sell_pause_execute`. Other reasons or flag false → propose. Model-originated stays propose.
4. **Elevated without allowlist: no execute.** Active `ir_elevated` / `break_glass` **without** `tool_allowlist_add` for that Hypermesh tool → propose / hold only. Profile alone is insufficient.
5. **`lease_stop` without incident: propose only.** Even under `break_glass`. Open `incident_id` required to execute.
6. **Drain order (H3).** Envelope with both execute-eligible tools → executor awaits `sell_pause` **result** before enqueueing any `lease_stop` for that device. Do not stop first. Different devices may parallelize.
7. **Plane down: Host-local still receipts.** `plane_reachable: false` → site still writes the receipt chain (propose / hold / notify). Owner-console apply uses Host H4 (`owner.sock` + token; `owner-effects.jsonl`). Do not skip the local receipt. Do not fake a CP job apply. Do not require the plane.
8. **PAIR unaffected.** PAIR down / preempt does not emit or execute `hypermesh.*`. Triage fallback unchanged.
9. **H3: both tools, same device.** Pause completes (`passed` + success `sell_state`) before any `lease_stop` enqueue for that device.
10. **H3: pause fail, non-`site_defense`.** Timeout / `passed=false` / still `selling` for `health_evacuate` (or any non-`site_defense` reason) → **no** automated stops; `hold_human` + notify; receipt records the pause failure.
11. **H3: pause fail, `site_defense` automated.** Rules-path `site_defense` with execute-eligible stops → stops **allowed**; pause failure **still recorded**. `preempt_mode` stays `drain` (not `hard`).
12. **H3: stop-only while `selling`, automated.** No `sell_pause` proposal, heartbeat `sell_state==selling`, automated actor → `hold_human` (optional inject pause propose). Not the pause-fail exception. Site flag `allow_stop_while_selling` default false.
13. **H3: stop-only while paused.** Heartbeat / last `sell_state` ∈ `{paused, off, n/a}` (or human-approved ticket) → allow execute-eligible stops.
14. **H3: pause OK, stop fails.** Sell stays **paused**. No auto-resume. Other `lease_stop`s in the batch continue. Notify on the failure.
15. **H3: no new Host job kind.** Drain uses only `sell_pause` + `lease_stop`. Executor must not invent a mega-job.
16. **H3: `hard` never selected without Host signal.** Host has no signal — v0 never selects `hard`, including the `site_defense` pause-fail exception.

Tests 1–16 are implementable against today's Host + CP (H1–H4 on Host **main**; Panopticon **[#37](https://github.com/FyberLabs/panopticon/pull/37)** `sell-pause-v0`, **[#40](https://github.com/FyberLabs/panopticon/pull/40)** site jobs, **[#41](https://github.com/FyberLabs/panopticon/pull/41)** site device `sell_state` read on **main**; H3 is Brewnix sequencing). Test 7 is the offline-dignity lock; Host H4 satisfies the owner-console path — do **not** require the site GET plane-up door when `plane_reachable: false`.

## Change control

Additive clarifications to this doc are fine. Weakening the matrix (auto `lease_stop` on `strict`, implying `hypermesh.*` from `rails_profile`, treating Path B as sell-pause, claiming Host already accepts iface args as local RPCs, selecting `preempt_mode=hard` without a Host signal, or turning drain into a Host mega-job) needs an explicit Chris amendment. The **H3 `site_defense` pause-fail exception** (allow stops after a failed pause on the automated rules path only) **is** that amendment — do not generalize it to other reasons. Semantic changes to locked files under `schemas/` remain **v1** or a Chris-approved schema amendment.

Implement policy / H3 sequencing against this file. Implement Host jobs in `FyberLabs/hypermesh-host` / Panopticon. Do not wait on a schema change.
