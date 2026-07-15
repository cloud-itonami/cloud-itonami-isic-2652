# ADR-0001: WatchOpsAdvisor ⊣ Watch & Clock Plant Operations Governor architecture

## Status

Accepted. `cloud-itonami-isic-2652` promoted from `:spec` to
`:implemented` in the `kotoba-lang/industry` registry, following the
verified fresh-scaffold protocol established by prior actors in this
fleet.

## Context

`cloud-itonami-isic-2652` publishes an OSS blueprint for watch/clock
**plant operations coordination** (production-batch product-type/
accuracy-test-result/quantity/defect-rate data logging, movement-
assembly/casing/regulation/testing-equipment maintenance scheduling,
safety-concern flagging, and outbound watch/clock shipment
coordination). Like every actor in this fleet, the blueprint alone is
not an implementation: this ADR records the governed-actor
architecture that promotes it to real, tested code, following the
same langgraph StateGraph + independent Governor + Phase 0->3 rollout
pattern established across the cloud-itonami fleet.

The closest domain analog is `cloud-itonami-isic-2710` (Manufacture of
electric motors, generators, transformers and electricity
distribution and control apparatus): both are back-office coordination
actors for a fixed processing PLANT with precision manufacturing/
assembly/test-bench equipment and a real physical safety dimension,
and both share the same four-op shape (`:log-production-batch`/
`:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment`)
and the same two-entity verified/registered gate structure (equipment
for maintenance scheduling, batch for shipment coordination), plus the
same domain-specific certification-authority permanent block shape
(2710 blocks self-issued electrical-safety marks; this build blocks
self-issued chronometer/accuracy marks). This build mirrors
`cloud-itonami-isic-2710`'s architecture closely but adapts the hazard
profile and equipment/product vocabulary to the watch/clock plant:
this vertical's central physical hazard is materials-safety (battery/
mercury-cell leakage or handling hazard in quartz movements), not
2710's high-voltage dielectric/hipot withstand-testing hazard (electric
shock / insulation-failure risk); its permanent equipment-actuation
block guards movement-assembly/casing/regulation/testing EQUIPMENT
(`:actuate-equipment?`) rather than winding/assembly/test-bench
EQUIPMENT; its production-batch record declares a `:product-type`
(closed set spanning mechanical-watch/automatic-watch/quartz-watch/
chronograph-watch/wall-clock/mantel-clock/movement) and an
`:accuracy-test-seconds-per-day` (the routine accuracy/rate-test
reading in seconds of deviation per day, plausibility-checked -60 to
+60 against general horological regulation practice and grounded
against ISO 3159's much tighter COSC chronometer band of -4/+6 s/day)
in addition to a `:defect-rate-percent`, rather than 2710's
`:dielectric-test-kv`; and its shipment quantity is tracked in
finished-unit UNITS (`:units`/`:quantity-units`/`:shipped-units`), the
same shape 2710 uses for finished motors/generators/transformers/
apparatus (counted, not weighed, for freight coordination) since
watches/clocks are likewise discrete counted units rather than a bulk
weight.

This vertical additionally has a DOMAIN-SPECIFIC permanent block in
the same shape 2710 needs (adapted, not copied verbatim): manufacture
of watches and clocks is subject to voluntary precision/accuracy
certification regimes (most notably COSC chronometer certification
under ISO 3159). This actor is never the certification authority — any
proposal (regardless of op) that declares `:issue-certification? true`
is a HARD, PERMANENT, unconditional block
(`watchmfg.governor/certification-authority-blocked-violations`), the
same "no phase, no human override" posture as the equipment-actuation
block.

This vertical has NO pre-existing `kotoba-lang/watchmfg`-style
capability library to wrap (verified: no such repo exists). This build
therefore uses self-contained domain logic — pure functions in
`watchmfg.registry` (equipment/batch verification, shipment-quantity
recompute, product-type validation, accuracy-test-result plausibility
validation, defect-rate plausibility validation) are re-verified
independently by the governor, the same "ground truth, not
self-report" discipline established across prior actors (most
directly `cloud-itonami-isic-2710`'s `elecequipmfg.registry`).

This blueprint's own `:itonami.blueprint/governor` keyword,
`:watch-clock-plant-operations-governor`, is grep-verified UNIQUE
fleet-wide (`gh search code "watch-clock-plant-operations-governor"
--owner cloud-itonami`, zero hits before this repo was created).

## Decision

### Decision 1: Self-contained domain logic (no external watch/clock-manufacturing capability library to wrap)

Unlike actors that delegate to pre-existing domain libraries, this
watch/clock vertical has NO pre-existing capability library to wrap.
The equipment/batch-verification / shipment-quantity / product-type /
accuracy-test-result / defect-rate validation functions live as pure
functions in `watchmfg.registry` and are re-verified independently by
`watchmfg.governor` — the same "ground truth, not self-report"
discipline established across prior actors (most directly
`cloud-itonami-isic-2710`'s `elecequipmfg.registry`).

### Decision 2: Coordination, not control — scope boundary at the back-office

This actor is **strictly back-office coordination** of watch/clock
plant operations. It does NOT:
- Control movement-assembly, casing, regulation, or testing equipment directly
- Make plant-safety or certification decisions (exclusive to the human plant supervisor / accredited certification body)
- Actuate movement-assembly/casing/regulation/testing equipment
- Self-issue a chronometer/accuracy certification mark (e.g. COSC under ISO 3159)

All proposals are `:effect :propose` only. The advisor proposes; the
governor validates; escalation paths funnel to human plant-supervisor
approval. This is not a replacement for the supervisor's authority or
the certification body's authority — it is a proposal-screening and
documentation layer.

**CRITICAL SAFETY BOUNDARY**: watch/clock manufacturing is a
safety-critical domain (battery/mercury-cell materials-safety hazard,
precision-defect risk, chronometer/accuracy certification, downstream
consumer-safety and precision-quality consequence). Safety-concern
flagging NEVER auto-commits. All safety concerns escalate immediately
to human review.

### Decision 3: Safety-concern escalation — always human sign-off

`:flag-safety-concern` (materials-safety concern, battery/mercury-cell
hazard concern, precision-defect concern, equipment-safety concern)
ALWAYS escalates, never auto-commits. This is not a "low-stakes
proposal" — it is a circuit-breaker that must reach human authority.

### Decision 4: Two independent verified/registered gates (equipment AND batch), not one

Like `cloud-itonami-isic-2710`, this vertical has TWO entity kinds
each gating a different op: `:schedule-maintenance` independently
verifies the referenced **equipment** unit's own `:verified?`/
`:registered?` fields; `:coordinate-shipment` independently verifies
the referenced **batch**'s own `:verified?`/`:registered?` fields.
Both are the same "plant/batch record must be independently
verified/registered before any action" HARD invariant applied to the
two distinct record kinds this domain actually has.
`:coordinate-shipment` additionally independently recomputes whether a
batch's own recorded shipped-to-date unit quantity plus the
proposal's own claimed unit quantity would exceed the batch's own
recorded production quantity — never taken on the advisor's
self-report.

### Decision 5: HARD invariants (no override)

Four HARD governor invariants (elaborated into twelve concrete checks
in `watchmfg.governor`, mirroring `cloud-itonami-isic-2710`'s own
elaboration of its HARD invariants into concrete checks) block
proposals and cannot be overridden by human approval:
1. Plant/batch record (equipment for maintenance, batch for shipment) must be independently verified/registered before any action is taken against it, and a shipment's quantity must independently recompute within the batch's own logged production quantity
2. Proposals must be `:effect :propose` only (never direct equipment control)
3. Direct movement-assembly/casing/regulation/testing-equipment control, equipment actuation, or self-issued chronometer/accuracy certification is permanently blocked
4. The op allowlist is closed — `:log-production-batch`/`:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` only

## Consequences

(+) Watch/clock plant operations back-office now has a documented,
governed, auditable coordination layer that funnels all decisions
through independent validation before human approval.

(+) The "coordination, not control" boundary is explicit in code: all
`:effect :propose`, all real-world actuation requires human plant-
supervisor sign-off, and no certification mark can ever be
self-issued.

(+) Scope is bounded and verifiable: four HARD invariants (elaborated
into twelve concrete governor checks) protect against scope creep into
unauthorized equipment operation, equipment actuation, or
certification self-issuance. Safety concerns are a circuit-breaker,
not a threshold.

(+) Safety-critical discipline is explicit: safety-concern flagging
cannot be rate-limited, suppressed, or auto-decided by phase gate.
Human review is mandatory.

(-) Still a simulation/proposal layer, not a real plant-operations
control system. Equipment actuation, line operation, and certification
issuance remain human-/institution-controlled via external channels.

(-) No integration with real plant-management databases (equipment
telemetry, batch tracking, freight dispatch, certification-body APIs)
— this is a standalone coordinator blueprint.

## Verification

- `cloud-itonami-isic-2652`: `clojure -M:test` green -- 76 tests
  containing 213 assertions, 0 failures, 0 errors (verified from an
  independent fresh clone; see the superproject ADR and
  `kotoba-lang/industry` registry entry for the exact re-verification
  output), `clojure -M:lint` clean (0 errors, 0 warnings), `clojure
  -M:dev:run` demo narrative exercises proposal submission,
  escalation, and every HARD-hold scenario directly (not-propose-
  effect, unknown-op, equipment-not-verified, batch-not-verified,
  shipment-quantity-exceeded, equipment-actuate-blocked,
  certification-authority-blocked, already-scheduled, invalid-
  product-type, invalid-accuracy-test-seconds-per-day, invalid-
  defect-rate).
- All source is `.cljc` (portable ClojureScript / JVM / nbb) — no
  JVM-only interop; the actor graph is invoked exclusively via
  `langgraph.graph/run*` (not `.invoke`, which is not cljs-portable).
- Audit ledger is append-only, all decisions are traced; every settled
  request (commit or hold) leaves exactly one ledger fact.
- `deps.edn` pins `io.github.kotoba-lang/langgraph` and
  `io.github.kotoba-lang/langchain` via `:local/root` directly in the
  top-level `:deps` (not only under a `:dev` alias), so a bare
  `clojure -M:test` resolves offline inside the monorepo checkout.
