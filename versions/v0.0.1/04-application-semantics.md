# 4. Application semantics

This section defines, precisely, what happens when a Requirement Type is extended and when a
Profile is applied. An implementation MUST produce the same effective Catalog as this algorithm
for the same input, regardless of whether resolution happens ahead-of-time or just-in-time
(see [6.4](06-architecture.md#64-ahead-of-time-vs-just-in-time-resolution)).

## 4.1 Reference resolution

A reference of the form `id@version` resolves to exactly one Requirement Type, Profile, or
Requirement Instance definition visible in the current Catalog's `requirementTypes`/`profiles`,
or in a configured external Type/Profile registry (see [section 8](08-configuration.md)). A
reference that resolves to zero or to more than one definition MUST be treated as a resolution
`error` Verification Result for every Binding that depended on it, not silently ignored.

## 4.2 Composition (`extends`)

Both Requirement Types and Profiles support `extends: [ref, ...]`.

1. Resolve each entry in `extends` recursively, left to right, before applying the current
   definition's own fields.
2. **Fields** merge by `name`. A field re-declared by a more specific definition (the one doing
   the `extend`-ing) overrides the inherited field entirely (type, `required`, `default`,
   `constraints`), rather than merging per-attribute.
3. **Checks** accumulate by `id` across the chain. A Check re-declared with the same `id`
   overrides the inherited Check entirely. A Check is never silently dropped by extension; it is
   only removed by an explicit suppression ([4.4](#44-suppression)) on the Instance.
4. Diamond inheritance (the same ancestor reached through two different `extends` paths) MUST
   resolve to a single copy of that ancestor's fields/checks — an implementation MUST deduplicate
   by resolved `id@version`, not re-apply it twice.
5. A cycle in `extends` MUST be rejected at load time with a load-time error; it MUST NOT be
   silently truncated.

The result of resolving `extends` for one Requirement Type or Profile is its **effective
definition**.

## 4.3 Applying a Profile

Given a Profile Application `{ profile: ref, parameterValues: {}, scopeOverride? }`:

1. Resolve `profile` to its effective definition (4.1, 4.2).
2. For every declared Parameter: if `parameterValues` supplies a value, use it; else if the
   Parameter has a `default`, use that; else if the Parameter is `required`, resolution fails
   with a load-time error naming the missing parameter.
3. For every Requirement Instance in the Profile's `instances`, substitute every
   `${parameterName}` placeholder appearing in `values` or `subjectSelector` with the resolved
   parameter value (string substitution; a placeholder with no matching parameter is a load-time
   error).
4. If `scopeOverride` is present, it replaces (not merges with) each resulting Instance's
   `subjectSelector`.
5. **Namespace** each resulting Instance's `id` as `<applicationId>/<instance.id>`, where
   `applicationId` is a stable id derived from the Profile ref and the Application's position
   (or an explicit `id` on the Application, if the format exposes one) — this is what allows the
   same Profile to be applied more than once (e.g. with different `sourceGlob` values) without
   the resulting Instance ids colliding.
6. Add every resulting, namespaced Instance to the effective Catalog.

Applying a Profile is **idempotent**: resolving the same Application twice (same profile ref,
same parameter values, same scope) MUST yield byte-for-byte identical effective Instances, so
that ahead-of-time and just-in-time resolution are observationally indistinguishable.

## 4.4 Suppression

A `checkOverrides` entry on a Requirement Instance MAY set `suppressed: true` to remove one
Check (by `id`) from that Instance's effective Checks.

- A Check with `suppressible: false` MUST NOT be suppressed; an attempt is a load-time error.
- Suppressing a Check whose effective `severity` is `error` or `blocker` MUST carry a non-empty
  `suppressionReason`; an implementation MUST reject the suppression otherwise. This is the
  standard's only mandatory audit-trail requirement: a `blocker`/`error` Check that does not run
  MUST say, in the data, why.
- A severity override (`checkOverrides[].severity` without `suppressed`) changes the Check's
  effective severity for that Instance only; it does not change the Requirement Type.

## 4.5 Binding to Subjects

For each effective Requirement Instance, its `subjectSelector` MUST be evaluated against the
Host's known Subject set (see [section 7](07-language-bindings.md) for how each language
enumerates Subjects) to produce zero or more `(Instance, Subject)` **Bindings**.

- An Instance whose selector matches zero Subjects produces no Bindings and MUST be reported as
  a distinct `unmatched` condition in the Verification Report ([section 5](05-verification-algorithm.md))
  — not silently dropped — since an unmatched selector is frequently an authoring mistake (a
  typo'd glob) that would otherwise hide a requirement that nobody is actually checking.
- An Instance with `status: draft` or `status: retired` MUST NOT produce Bindings; only
  `approved` and `deprecated` Instances bind (a `deprecated` Instance still binds, so its decay
  stays visible, but SHOULD lower its effective severity by one level unless overridden).
