# LOCK — 2026-09-06

**Status:** working contract  
**Schemas:** `fyber.inference_iface/v0`, `fyber.receipt/v0`, `fyber.feature_bundle/v0`, `fyber.common/v0`  
**Owner decision:** Chris Hamilton  
**Recorded by:** First Principles Reviewer/Deconstructor

## Frozen

- Envelope = judgment + typed `proposals` only
- Allowlisted tools only (see README)
- Execute path = policy + typed actuators; LLM never executes
- Receipts = digests / policy / effects / hash chain; no prompts or payloads
- Feature bundles redacted and fixed-schema
- Offline-first

## Change control

- Additive optional fields: OK behind a feature flag without renaming `schema`
- Renames, removals, new execute tools, or semantic changes to digests/integrity: **v1** or explicit Chris amendment
