# 9. Conformance and testing

## 9.1 Conformance levels

| Level | Requires |
|---|---|
| L0 | Reads/validates the data shapes in [section 3](03-core-model.md) only. No verification. |
| L1 | L0 + a Runtime implementing [section 5](05-verification-algorithm.md) end to end. |
| L2 | L1 + a Design-Time API per [section 6](06-architecture.md), independently shippable. |
| L3 | L2 + passes the standard's cross-language conformance test vectors ([7.1](07-language-bindings.md#71-conformance-test-vectors)) with byte-for-byte identical Verification Reports. |

An implementation MUST state the Conformance Level it claims, and MUST NOT claim L2 or L3
without the corresponding artifacts actually being independently shippable / actually passing
the vectors — an unexecuted check is a gap, not an implicit pass.

## 9.2 Testing mandate

This standard does not relax the monorepo's own testing rules; it makes them explicit
conformance requirements for Requirements-As-Code implementations specifically, per the root
[`AGENTS.md`](../../../../AGENTS.md) property-testing/fuzzing/mutation-testing section:

- **Property-based tests** MUST cover: `extends` resolution (4.2) — including the diamond and
  cycle cases — and Profile Application (4.3), asserting the idempotency property stated there
  (applying the same Application twice yields identical effective Instances) and the AOT/JIT
  equivalence property stated in
  [6.4](06-architecture.md#64-ahead-of-time-vs-just-in-time-resolution).
- **Fuzzing** MUST exercise the Catalog/Profile/Requirement-Type parser against malformed,
  truncated and adversarially nested (`extends` cycles, missing parameters, unresolved refs)
  input, asserting that every malformed input produces a reported load-time error — never a
  crash, never a silently-empty Catalog.
- **Mutation testing** MUST be run against the verification algorithm ([section 5](05-verification-algorithm.md))
  to confirm that its test suite's assertions actually detect a changed outcome (e.g. a mutated
  severity comparison, a flipped `fail-fast` branch) rather than merely re-stating the
  implementation. Surviving mutants MUST be investigated and classified (equivalent mutant,
  missing assertion, invalid mutant) per the monorepo's mutation-testing rules.
- Ordinary example tests alone do not satisfy conformance at any level above L0.

## 9.3 Required test run visibility

Any test or mutation campaign run against a Requirements-As-Code implementation follows the
monorepo's test-run rules: a visible, dedicated monitoring window for the run, a bordered table
summary, and a terminal-state report (success, failures, crash, timeout) — silence is never the
only sign of an outcome, for the same reason an `unmatched` selector
([4.5](04-application-semantics.md#45-binding-to-subjects)) is reported rather than dropped.

## 9.4 Execution conformance

[Section 11](11-execution-models.md#117-execution-conformance) defines an orthogonal tag set —
`Embeddable`, `Serviceable`, `Containerizable` — reported alongside the Conformance Level above.
A binding states both: the Level describes how much of the model it implements, the tag set
describes where it can run.

## 9.5 Checklist

- [ ] Requirement Type / Requirement Instance / Profile / Catalog schemas (section 3) validated.
- [ ] `extends` resolution: linear, diamond and cycle-rejection cases tested.
- [ ] Profile Application: parameter substitution, namespacing, idempotency tested.
- [ ] Suppression rule (4.4): `suppressible: false` and missing-reason cases rejected.
- [ ] Verification algorithm (section 5): all six `outcome` values reachable and tested.
- [ ] Severity threshold and `fail-fast`/`collect-all` both tested.
- [ ] Contracts/Runtime/Design-Time dependency direction enforced (no Runtime→Design-Time edge).
- [ ] Configuration (section 8) read identically from the default file and from `RAC_CONFIG`.
- [ ] Property, fuzz and mutation campaigns run and their results retained, per 9.2.
- [ ] Conformance Level claimed matches what was actually built and verified.
- [ ] `executionModel: embedded` requires no network dependency.
- [ ] Service health/ready endpoints respond correctly before and after catalog registration.
- [ ] Service handles a stop signal within `shutdownGraceSeconds` without dropping an in-flight report.
- [ ] Container image passes its own `HEALTHCHECK` and exposes `bindPort`.
- [ ] Embedded and Service execution of the same Catalog produce identical Verification Reports.
- [ ] Execution Conformance tags claimed (section 11.7) match what was actually built and verified.
