# cloud-itonami-isic-2652: Manufacture of watches and clocks

Open Business Blueprint for **ISIC 2652**: manufacture of watches and clocks — an autonomous "actor" (LLM advisor behind an independent Governor, langgraph-clj StateGraph, append-only audit ledger) that coordinates back-office **watch/clock-plant operations**: production-batch data logging (product-type/accuracy-test-result/quantity/defect-rate), movement-assembly/casing/regulation/testing-equipment maintenance scheduling, safety-concern flagging, and outbound product shipment coordination.

This repository designs a forkable OSS business for watch/clock-plant
operations: run by a qualified operator so a plant keeps its own
operating records instead of renting a closed SaaS.

## Scope: plant operations coordination, not movement-assembly/testing-line control

ISIC 2652 covers the **manufacturing plant** that assembles watch and clock movements (mechanical and quartz), cases them, regulates them, and tests them (including accuracy/rate testing against a reference timebase) before shipment. This actor coordinates the back-office record keeping around that plant — it never touches the movement-assembly/casing/regulation/testing equipment directly, and it is never a chronometer/accuracy certification authority (e.g. COSC compliance marks under ISO 3159).

## What this actor does

Proposes **plant operations coordination**, not equipment operation:
- `:log-production-batch` — movement-assembly/casing batch, output-quality/accuracy-test-result data logging (administrative, not an operational decision)
- `:schedule-maintenance` — movement-assembly/casing/regulation/testing-equipment maintenance scheduling proposal
- `:flag-safety-concern` — surface a materials-safety (battery/mercury-cell hazard)/precision-defect concern (always escalates)
- `:coordinate-shipment` — outbound watch/clock shipment coordination proposal

## What this actor does NOT do

**CRITICAL SCOPE BOUNDARY — this is a safety-critical domain**
(movement-assembly/casing/regulation/testing line equipment, battery/
mercury-cell materials-safety hazard, chronometer/accuracy
certification, downstream consumer-safety and precision-quality
consequence):

- Does NOT control movement-assembly, casing, regulation, or testing equipment directly
- Does NOT make plant-safety or certification decisions (that's the plant supervisor's / certification body's exclusive human/institutional authority)
- Does NOT actuate movement-assembly/casing/regulation/testing equipment (human plant supervisor decides)
- Does NOT self-issue a chronometer/accuracy certification mark (e.g. COSC under ISO 3159 — the accredited certification body's exclusive authority — a PERMANENT, unconditional block)
- ONLY proposes/coordinates operations back-office; all actuation and certification requires explicit human/institutional authority
- Safety-concern flagging ALWAYS escalates — never auto-decided, no confidence threshold or phase below escalation

## Architecture

Classic governed-actor pattern (`watchmfg.operation/build`, a langgraph-clj StateGraph):
1. **`watchmfg.advisor`** (sealed intelligence node, `WatchOpsAdvisor`): proposes decisions only, never commits
2. **`watchmfg.governor`** (independent, `Watch & Clock Plant Operations Governor`): validates against domain rules, re-derived from `watchmfg.registry`'s pure functions and `watchmfg.store`'s SSoT -- never trusts the advisor's own self-report
   - HARD invariants (always `:hold`, no override):
     - Plant/batch record must be independently verified/registered (`:verified?` AND `:registered?`) before any action is taken against it (equipment before maintenance scheduling, batch before shipment coordination)
     - The request's own `:effect` must be `:propose` (never a direct-write bypass)
     - `:op` must be in the closed four-op allowlist
     - The proposal's own `:effect` must be one of the four propose-shaped effects (no direct movement-assembly/casing/regulation/testing-equipment control)
     - Directly actuating movement-assembly/casing/regulation/testing equipment (`:actuate-equipment? true`) is a PERMANENT, unconditional block
     - Self-issuing a chronometer/accuracy certification mark (`:issue-certification? true`, any op) is a PERMANENT, unconditional block
     - A shipment may not push a batch's own recorded shipped quantity past its own logged production quantity (independently recomputed)
     - No double-scheduling the same maintenance record
     - No fabricated `:product-type` value on a production-batch patch
     - No physically implausible `:accuracy-test-seconds-per-day` value on a production-batch patch
     - No physically implausible `:defect-rate-percent` value on a production-batch patch
   - ESCALATE (always human sign-off, overridable by a human):
     - `:flag-safety-concern` always escalates, regardless of confidence
     - Low-confidence proposals
3. **`watchmfg.phase`** (Phase 0->3 rollout): `:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` are NEVER in any phase's `:auto` set (permanent, matching the governor's own posture); only `:log-production-batch` may auto-commit at phase 3 when clean
4. **`watchmfg.store`** (append-only audit ledger + SSoT): a single `MemStore` backend behind a `Store` protocol (see ns docstring for why a second Datomic-backed backend is out of scope for this build)

## Development

```bash
# Run tests (top-level deps.edn already pins langgraph+langchain local/root)
clojure -M:test

# Run tests via the workspace :dev override alias (equivalent, kept for sibling-repo parity)
clojure -M:dev:test

# Run the demo
clojure -M:dev:run

# Lint
clojure -M:lint
```

## Status

`:implemented` — `governor.cljc`/`store.cljc`/`advisor.cljc`/`registry.cljc` + `deps.edn` complete the module set; tests green, demo runnable, langgraph-clj integration verified.

## License

AGPL-3.0-or-later
