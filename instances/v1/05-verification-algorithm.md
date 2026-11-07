# 5. Verification algorithm

This section defines, abstractly, how a Runtime turns a resolved effective Catalog into a
Verification Report. Every Runtime binding (PowerShell, C#, Rust, C++) MUST implement exactly
this algorithm; a binding MAY parallelize steps 2–3 across Bindings, but MUST NOT change their
observable outcome.

## 5.1 Inputs

- The effective Catalog (section 4), already bound to Subjects
  ([4.5](04-application-semantics.md#45-binding-to-subjects)).
- A severity threshold (section 8) and a short-circuit policy: `fail-fast` or `collect-all`
  (default `collect-all`).

## 5.2 Steps

1. For each Binding `(Instance, Subject)`, in Catalog order:
   1. Evaluate each of the Instance's effective Checks (4.2, 4.4), in the order they appear on
      the (possibly extended) Requirement Type, skipping any Check removed by suppression.
   2. A Check's `kind` determines how its `expression`/data is evaluated:
      - `expression`: a boolean-valued predicate over `(subject, fields, context)` — the
        expression language is implementation-defined per binding (see section 7) but MUST be
        pure (no I/O, no mutation of the Subject) and MUST terminate; a binding MUST enforce a
        configurable evaluation timeout and MUST report a timeout as `error`, never as `pass`.
      - `regex`: match a named property of the Subject against `expression` as a regular
        expression.
      - `schema`: validate a named property of the Subject against a JSON Schema given in the
        Check's data.
      - `external`: delegate to a named, host-registered callback (e.g. a C# static analyzer
        rule, a PowerShell script block); the standard does not constrain its implementation,
        only its contract: it MUST return one of `pass`/`fail`/`warn`/`error` and MUST NOT let
        an unhandled exception escape past the Runtime boundary — an unhandled exception MUST be
        captured and reported as `error`, not allowed to abort the whole run.
   3. Each Check evaluation produces one Verification Result:
      `{ instanceId, subjectId, checkId, outcome, severity, message, evidence?, timestamp }`.
   4. If the short-circuit policy is `fail-fast` and this Result's `outcome` is `fail` at or
      above the configured severity threshold, stop evaluating remaining Checks for this run
      entirely (not just this Binding) and proceed to 5.3.
2. Collect every Verification Result (and every `unmatched` condition from 4.5) into the
   Verification Report.

## 5.3 Verification Report

```json
{
  "$id": "urn:frenchexdev:rac:v1:verification-report",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["catalogId", "catalogVersion", "startedAt", "finishedAt", "results"],
  "properties": {
    "catalogId": { "type": "string" },
    "catalogVersion": { "type": "string" },
    "startedAt": { "type": "string", "format": "date-time" },
    "finishedAt": { "type": "string", "format": "date-time" },
    "results": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["instanceId", "checkId", "outcome", "severity"],
        "properties": {
          "instanceId": { "type": "string" },
          "subjectId": { "type": "string" },
          "checkId": { "type": "string" },
          "outcome": { "enum": ["pass", "fail", "warn", "skipped", "error", "unmatched"] },
          "severity": { "$ref": "urn:frenchexdev:rac:v1:severity" },
          "message": { "type": "string" },
          "evidence": {},
          "timestamp": { "type": "string", "format": "date-time" }
        }
      }
    }
  }
}
```

## 5.4 Severity and thresholds

Severities order as `info < warning < error < blocker`. A Host configures a severity threshold
(section 8); the Runtime's exit/return contract is:

- Any Result with `outcome: fail` or `outcome: error` at or above the threshold MUST cause the
  Runtime's top-level verification call to report overall failure (a non-zero process exit code
  for a CLI host; a failed `Task`/`Result`/equivalent for a library call — never a silently
  swallowed error).
- Results below the threshold, and every `pass`/`skipped`/`warn`, MUST still appear in the
  report; "below threshold" changes whether the run fails overall, never whether a Result is
  recorded. Silence must never be the only evidence of an outcome.
- `unmatched` ([4.5](04-application-semantics.md#45-binding-to-subjects)) is always reported and
  SHOULD be surfaced distinctly in any human-facing summary, so an author notices a selector
  that matches nothing.
