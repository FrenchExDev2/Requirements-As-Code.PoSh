# 1. Overview

## 1.1 Problem statement

Requirements — architectural rules, security controls, coding standards, operational SLOs,
compliance obligations — are usually written as prose in wikis or review checklists. Prose
requirements cannot be versioned precisely, composed, parameterized, or checked by a machine;
every project re-derives its own ad hoc enforcement (a linter rule here, a manual review step
there), and drift between "what the document says" and "what actually runs" is discovered late,
if at all.

Requirements-As-Code (RaC) treats a requirement the same way this monorepo already treats
behavior: as versioned, typed data with a defined contract, generated into language-specific
artifacts, and verified automatically rather than by manual review.

## 1.2 Goals

RaC version 1 MUST allow an implementation to:

1. Define a **Requirement Type** once — the schema for one kind of requirement (its fields and
   the checks that decide whether an instance of it is satisfied) — and reuse it across
   projects and languages.
2. Bundle Requirement Types and preconfigured **Requirement Instances** into a named,
   versioned, parameterizable **Profile**, and **apply** that profile to a project or module
   with a single declarative action.
3. Generate, from Requirement Types/Instances/Profiles, two distinct kinds of language
   artifacts: a **Runtime** artifact (verifies real subjects while the system operates or in
   CI) and a **Design-Time API** artifact (assists authors in creating and validating
   requirement data, with no production dependency on it).
4. Drive all of the above — which catalogs, which profiles, which severity threshold, which
   verification trigger — by **configuration**, identically, from PowerShell, C#, Rust and C++
   hosts, per [section 8](08-configuration.md).
5. Verify requirements **at runtime**, producing a structured, machine-readable report rather
   than a pass/fail boolean, per [section 5](05-verification-algorithm.md).

## 1.3 Non-goals

RaC version 1 does **not**:

- Replace issue trackers, ticketing systems or review workflows — it produces verification
  evidence that such systems MAY consume.
- Mandate a storage technology for catalogs. Section 8 defines how a catalog source is
  *referenced*, not where it must live (files, a database, an HTTP endpoint are all valid).
- Define a visual authoring UI. It defines the data and code contracts a UI could be built on.
- Replace language-level contract systems (code contracts, schema validators, static analyzers).
  A Check MAY delegate to one of these instead of reimplementing its logic.

## 1.4 Design principles

These principles are the monorepo's own [`AGENTS.md`](../../../../AGENTS.md) SOLID/DRY/modular
and async-first rules, applied specifically to this standard:

- **Single responsibility per artifact.** Contracts define shape. Runtime verifies. Design-Time
  assists authors. No artifact does more than one of these jobs (see [section 6](06-architecture.md)).
- **Depend on stable contracts, not on siblings' internals.** Design-Time MAY depend on
  Contracts and on Runtime's read-only result model; Runtime MUST NOT depend on Design-Time.
- **DRY at the semantic level.** The data model and the verification algorithm are specified
  exactly once, in this document set. A language binding MUST be a faithful mapping of that one
  semantics, never a parallel redefinition of it.
- **Configuration over code.** Changing which catalogs/profiles/thresholds apply MUST be a
  configuration change ([section 8](08-configuration.md)), not a recompile.
- **Async by default.** Any Runtime or Design-Time operation that performs I/O (loading a
  catalog, resolving a type registry, writing a report) MUST expose an asynchronous entry point
  in languages where that distinction exists (C#, Rust); a synchronous convenience wrapper MAY
  also be offered, but MUST NOT be the only option, consistent with the monorepo-wide async
  preference.

## 1.5 Audience

- **Standard implementers** building a Contracts/Runtime/Design-Time package set for a language.
- **Requirement authors** writing Requirement Types, Profiles and Instances for a project.
- **Tool builders** integrating verification reports into CI, dashboards or audit pipelines.
