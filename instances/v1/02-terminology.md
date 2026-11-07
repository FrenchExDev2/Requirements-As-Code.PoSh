# 2. Terminology

This section is normative. Every other document in this standard uses these terms exactly as
defined here.

**Requirement Type**
: A reusable, named, versioned schema for one kind of requirement. Declares the **Fields** an
  instance of this type must carry, the **Checks** that decide whether an instance is satisfied,
  and a default **Severity**. A Requirement Type MAY `extend` one or more other Requirement
  Types (see [4.2](04-application-semantics.md#42-composition-extends)).

**Field**
: A single named, typed data element on a Requirement Type (e.g. `maxLineLength: integer`).

**Check**
: A single, named, independently evaluable rule attached to a Requirement Type. A Check has a
  `kind` (e.g. `expression`, `regex`, `schema`, `external`), the data that rule needs, a
  human-readable `message` template for failures, and a `severity`.

**Severity**
: An ordered level (`info` < `warning` < `error` < `blocker`, see
  [5.4](05-verification-algorithm.md#54-severity-and-thresholds)) attached to a Check or
  overridden on a Requirement Instance.

**Requirement Instance** (**Instance**)
: Concrete data conforming to one Requirement Type: field values, optional Check overrides, a
  **Subject Selector**, a lifecycle `status` (`draft`, `approved`, `deprecated`, `retired`) and
  free-form evidence/metadata.

**Profile**
: A named, versioned, parameterizable bundle of Requirement Types and/or preconfigured
  Requirement Instances, meant to be **applied** as a unit (e.g. "apply the
  `SOLID-DotNet-Library` profile to this package"). A Profile MAY declare **Parameters** with
  defaults, substituted into its bundled Instances at application time. A Profile MAY `extend`
  (include) other Profiles.

**Apply** / **Application**
: The act of resolving one Profile (with concrete parameter values and an optional scope
  override) into a set of concrete, bound Requirement Instances, and materializing them into a
  target **Catalog**. Defined precisely in [section 4](04-application-semantics.md).

**Subject**
: The real thing a Requirement Instance is checked against: a file, a module, a package, a
  running process, a configuration value, an infrastructure resource, a document — anything a
  Check's `kind` knows how to inspect.

**Subject Selector**
: Data on a Requirement Instance that identifies which Subject(s) it binds to (e.g. a glob, a
  tag match, an explicit id list).

**Binding**
: One resolved `(effective Requirement Instance, Subject)` pair, ready to be evaluated.

**Catalog**
: The materialized, versioned set of effective Requirement Instances — from standalone
  Instances plus every Profile Application — for one scope/namespace. The Catalog is what a
  Verifier actually loads; it is the output of applying Profiles, not raw authoring input.

**Verification Result**
: The outcome (`pass`, `fail`, `warn`, `skipped`, `error`, `unmatched`) of evaluating one
  Binding's Checks, with messages and evidence.

**Verification Report**
: The aggregate of every Verification Result produced by one verification run.

**Runtime** (artifact)
: The generated/compiled component that loads a Catalog and evaluates Bindings against live
  Subjects, producing a Verification Report. Has no dependency on Design-Time.

**Design-Time API** (artifact)
: The generated/compiled component exposing a strongly-typed authoring surface — model types,
  builders, a Profile/Type registry client, and (optionally) an ahead-of-time Apply operation —
  used by tools, build steps and IDEs, never by the running production system.

**Contracts** (artifact)
: The language-neutral model types and interfaces that both Runtime and Design-Time depend on,
  defined once per language family to avoid duplicating the model (see [section 6](06-architecture.md)).

**Generator**
: The standard-conformant tool that reads Requirement Types, Profiles and Catalogs and emits a
  language's Contracts/Runtime/Design-Time artifacts.

**Host**
: The process (a PowerShell session, a .NET application, a Rust binary, a C++ program) that
  loads a Runtime artifact and configuration and triggers verification.

**Conformance Level**
: The degree to which an implementation satisfies this standard, `L0`–`L3`, defined in
  [section 9](09-conformance-and-testing.md).
