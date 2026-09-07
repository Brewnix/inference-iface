# SociACL IR binding v0

**Status:** working spec — **LOCKED 2026-09-07** (Chris; after SociACL Dev Bot inventory + mask pressure-test)  
**Schemas:** pin this repo (`fyber.inference_iface/v0`, `fyber.receipt/v0`, `fyber.feature_bundle/v0`, `fyber.common/v0`) — **no v0 schema break**  
**Binding:** Brewnix IR objects + masks only (docs-first; **not** a file under `schemas/`)  
**Source of truth:** [FyberLabs/SociACL](https://github.com/FyberLabs/SociACL) — Check, Remint, `delegate` / light `see`. This repo is the **Brewnix binding** only. Do not fork primitives here.  
**Companion:** [fyber.privilege_grant v0](privilege-grant-v0.md) (body stays Brewnix; SociACL authorizes minting) · [fyber.auditor API v0](fyber-auditor-api-v0.md) (ticket resolve on `:ir`) · [Hypermesh preempt v0](hypermesh-preempt-v0.md) (`:host` execute when Host exists; Checkout **not** a consumer) · [fyber.incident binding v0](incident-binding-v0.md) (overlay case; per-incident ACL objects **deferred**) · [Zero-LLM OPNsense detect → block → receipt loop](zero-llm-opnsense-loop.md) (contain never Checks per event)

Goal: map SociACL **who may act** onto Brewnix IR objects. Brewnix policy still decides **what** act — `execute` vs `propose` / `hold_human`. IR keep-operating is Check + **delegate** (`read` \| `write` \| `execute`, optional `until`) and light **see**. Not Elect, not wills, not Case C.

**Home:** this repo holds the binding (MIT / OSS intent). The crate remains the primitive SoT ([delegate PR #11](https://github.com/FyberLabs/SociACL/pull/11), [light Check PR #13](https://github.com/FyberLabs/SociACL/pull/13) / [`docs/s3rch-check.d.ts`](https://github.com/FyberLabs/SociACL/blob/master/docs/s3rch-check.d.ts)). Site JSON Schemas stay locked; do not add SociACL fields to `schemas/`.

## Goal / non-goals

| Goal | Meaning |
|------|---------|
| Who vs what | SociACL Check answers who may act. Brewnix policy answers execute vs propose. |
| Keep-operating only | IR uses Check + `delegate` (mask + optional `until`) / light `see`. Owner stays owner. |
| Logical ACL nodes | Standing `site:{site_id}`, `:ir`, and `:host` (when Host exists). |
| Re-Check at act | Session cookie / prior zookie alone is not enough to resolve or mint. |
| Cancel is down | `undelegate` / privilege-down is immediate. Contain does not wait on privilege-up or Remint. |

| Non-goal | Why |
|----------|-----|
| SociACL primitive crate | [FyberLabs/SociACL](https://github.com/FyberLabs/SociACL) is SoT. This file binds. |
| `break_glass` verb in SociACL | None today. Break-glass approve is a **Brewnix** gate (owner + execute on `:ir`). |
| Owner-console or site-token type in SociACL | None today. Owner console is a Host/Brewnix product. Machine site tokens stay Panopticon-parallel. |
| Elect / wills / Case C for IR | Keep-operating only. A live `delegate` is not an election. |
| Per-incident ACL objects | Deferred. [Incident](incident-binding-v0.md) stays a Brewnix overlay. |
| Hop / URL / auditor handoff as a grant | Hint only. Dest re-Checks. |
| Hypermesh Checkout as consumer | [Preempt](hypermesh-preempt-v0.md) is Brewnix → Host jobs. Not a SociACL grant consumer. |
| JSON Schema under `schemas/` | Implement against this file. |
| Rust crate in the browser | s3r.ch pattern: copy/re-type light Check. Hopcap **1**. |

## Axioms

1. **SociACL Check = who may act. Brewnix policy = what act** (`execute` vs `propose` / hold).
2. **IR keep-operating = Check + `delegate`** (`read` \| `write` \| `execute`, optional `until`) **/ light `see`.** Not Elect, not wills, not Case C.
3. **No SociACL `break_glass` verb.** No owner-console type and no site-token type in SociACL today.
4. **Hop / URL / auditor handoff is never a grant.** Dest re-Checks. A flash or hint does not mint `delegate`.
5. **`fyber.privilege_grant/v0` body stays Brewnix.** SociACL authorizes **minting** (who may approve). The grant object is not a SociACL `delegate`.
6. **Zero-LLM contain / expiry = no per-event Check.** Standing owner policy on the site box. Do **not** call Check on each `firewall.block_ip` / `firewall.unblock_ip`.
7. **Privilege-down cancel is immediate. Privilege-up may wait.** Do **not** block contain on a pending Remint or privilege-up delay.
8. **Light surface / hopcap 1.** s3r.ch pattern: TypeScript light Check. Do **not** assume the Rust crate in the browser. Hopcap **> 1** is out of v0.

## Objects (logical ACL nodes)

Logical names only. Not a schema field. Not a file under `schemas/`.

| Object | Role |
|--------|------|
| `site:{site_id}` | Standing owner possession. |
| `site:{site_id}:ir` | IR keep-operating (auditor tickets, non-break-glass grant mint). |
| `site:{site_id}:host` | Host preempt console (**when Host exists**). |
| Per-incident objects | **Deferred.** Do not mint `site:{site_id}:incident:{incident_id}` in v0. |

`site_id` is the same token as envelopes / grants / auditor scope.

## Masks

`see` is the light Check consume verb (maps to dest Check `read` on the s3r.ch contract). `delegate` carries `read` \| `write` \| `execute` and optional `until`. Missing hop does not fail Check. A hop alone does not mint.

| SociACL | Object | Brewnix may |
|---------|--------|-------------|
| `see` **or** `delegate` **`read`** | `:ir` | View **redacted** tickets / receipts. |
| `delegate` **`write`** | `:ir` | **Annotate only.** Not state-changing resolve. Not mint. |
| `delegate` **`execute`** | `:ir` | Resolve held tools **and** mint / approve **non-`break_glass`** [`privilege_grant`](privilege-grant-v0.md)s. |
| `delegate` **`execute`** **and** principal is **owner** | `:ir` | Approve **`break_glass`**. Owner gate is **Brewnix**, not a SociACL verb. Enterprise = **two** owner Checks (Brewnix dual-control). |
| `delegate` **`execute`** | `:host` | Owner-console preempt ack **when Host exists** ([preempt](hypermesh-preempt-v0.md)). |

**Write-only cannot resolve.** `delegate write` without `execute` is annotate. Ticket `POST …/resolve` and grant mint require `execute` on `:ir` (plus the owner gate for `break_glass`).

### Out of SociACL v0

**Machine site tokens** (auditor notify / ack): **not** a SociACL type today. Stay **Panopticon-parallel** until a device-node `delegate` exists. Do not pretend the Bearer site token is `delegate` on `:ir`.

## Standing vs ephemeral

Owner **Remint**s time-bounded `delegate`s on `:ir` / `:host` (optional `until`). Owner stays owner. Remint refreshes only if the current ACL already names the principal.

**Cancel** (`undelegate`) = privilege-down **immediate**. Object version bumps. Resolve rights are gone on the next Check. Do not honor a cached session allow.

Privilege-up (joint statement + delay) **may wait**. Contain / expiry **must not** wait on that delay or on a pending Remint (axiom 7).

## `privilege_grant` ≠ `delegate`

| | `fyber.privilege_grant/v0` | SociACL `delegate` |
|--|----------------------------|--------------------|
| Job | Time-bounded **policy** elevation for one `(site_id, incident_id)` | Keep-operating **who-may-act** on an ACL object |
| Body | Five ask kinds + `rails_profile` + TTL ([grant spec](privilege-grant-v0.md)) | Mask `read` \| `write` \| `execute`, optional `until` |
| Mint | Human / dual-control; SociACL Check authorizes the mint | Owner-only `Plane::delegate` (crate) |
| Empty | `asks: []` → refuse (use a ticket) | Not a policy catalog |

A ticket [resolve](fyber-auditor-api-v0.md) may mint a grant. That resolve still **re-Checks** `execute` on `:ir` at resolve time. Approving `break_glass` additionally requires the Brewnix owner gate (two Checks on enterprise).

## Light surface / hopcap

Browser / plane UI follows the s3r.ch consume contract: `CHECK(see, object, accessor)` at now, hopcap **1**, jointly stated grants, revoke immediate, URL / hop handoff is a **hint**, dest re-authorizes before act.

- Copy or re-type [`docs/s3rch-check.d.ts`](https://github.com/FyberLabs/SociACL/blob/master/docs/s3rch-check.d.ts). Do **not** `npm install sociacl`. Do **not** import the Rust crate, NAPI, or WASM on the Next / auditor light path.
- Auditor handoff URL is the same class as a Gun `HandoffHint`: untrusted; never a grant.

## Do not assume

From the SociACL Dev Bot inventory. These are **not** v0.

- Hop / hint / URL fetch is a grant
- Elect / wills / Case C for IR
- Next (or the auditor UI) imports the Rust crate
- Hopcap **> 1**
- Instant privilege-up everywhere
- Hypermesh Checkout as a SociACL consumer of these grants
- Owner-console product or `break_glass` verb **in SociACL**

## Acceptance tests

1. **`delegate execute` on `:ir` can resolve a ticket.** Same principal **cannot** approve `break_glass` without the Brewnix **owner** gate (enterprise: two owner Checks).
2. **Write-only cannot resolve.** `delegate write` (no `execute`) may annotate; `POST …/resolve` and grant mint are denied.
3. **Re-Check at resolve time.** A prior session / stale zookie is insufficient. Check `execute` on `:ir` at the resolve / mint instant.
4. **Cancel delegate → immediate loss of resolve rights.** After `undelegate` (or `until` elapsed), the next resolve / mint is denied. Do not serve a cached allow.
5. **Contain path never calls Check per event.** [Zero-LLM](zero-llm-opnsense-loop.md) `firewall.block_ip` / expiry unblock run on standing owner policy. Pending Remint must not delay the block.
6. **`privilege_grant` object ≠ SociACL `delegate`.** Minting a grant requires Check; the stored `fyber.privilege_grant/v0` body is not a `delegate` edge and does not by itself allow ticket resolve.

Shape reference for the object/mask table: `examples/sociacl-ir-binding.example.json`.

## Change control

Additive clarifications to this doc are fine. A SociACL `break_glass` verb, owner-console or site-token **type in the crate**, per-incident ACL objects, hopcap > 1, Elect/wills/Case C on IR, treating hop/URL as a grant, Hypermesh Checkout as a consumer of these grants, or a `schemas/sociacl*.json` file need an explicit Chris amendment. Semantic changes to locked files under `schemas/` remain **v1** or a Chris-approved schema amendment.

Implement the binding against this file. Implement primitives in [FyberLabs/SociACL](https://github.com/FyberLabs/SociACL). Do not wait on a schema file under `schemas/`.
