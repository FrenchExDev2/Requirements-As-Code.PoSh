# Requirements-As-Code — Standard, Version 1 (Draft)

| | |
|---|---|
| Status | Draft |
| Version | 1.0.0 |
| Editor | FrenchExDev |
| Date | 2026-11-07 |

## Abstract

This document defines version 1 of the Requirements-As-Code (RaC) standard: a data model and a
set of contracts for expressing requirements as versioned, typed, machine-checkable data (a
**Requirement Instance** of a **Requirement Type**, bundled into a named, applicable
**Profile**), for compiling that data into language-specific **Design-Time** and **Runtime**
artifacts, and for verifying requirements against real subjects while a system runs.

The standard is language-neutral. Conforming bindings for PowerShell, C#, Rust and C++ are
defined in [`07-language-bindings.md`](07-language-bindings.md); the same abstract model and
algorithm MUST produce the same verification outcome regardless of which binding evaluates it.

## Conventions used in this document

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are to be
interpreted as described in RFC 2119: they indicate conformance requirements, not merely a
stylistic preference.

## Table of contents

1. [Overview](01-overview.md) — goals, non-goals, design principles
2. [Terminology](02-terminology.md) — defined terms used throughout the standard
3. [Core model](03-core-model.md) — Requirement Type, Requirement Instance, Profile, Catalog
4. [Application semantics](04-application-semantics.md) — extension, composition, applying a Profile, binding to subjects
5. [Verification algorithm](05-verification-algorithm.md) — how a Binding becomes a pass/fail result
6. [Architecture: Contracts / Runtime / Design-Time](06-architecture.md) — the three-artifact shape every binding MUST produce
7. [Language bindings](07-language-bindings.md) — PowerShell, C#, Rust, C++
8. [Configuration](08-configuration.md) — the one configuration schema every binding reads
9. [Conformance and testing](09-conformance-and-testing.md) — conformance levels, required test strategy
10. [Design patterns and strategies](10-design-patterns-and-strategies.md) — the POSA/GoF patterns sections 3–6 already realize, and which known problems they avoid
11. [Execution models and deployment](11-execution-models.md) — embedded, VM-hosted and containerized Runtime execution, and the Verification Service API
12. [Observability and profiling](12-observability-and-profiling.md) — diagnostics events, tracing, metrics, and the profiling-must-not-change-outcomes rule
13. [Configuration precedence and levels](13-configuration-precedence-and-levels.md) — how data-level overrides and run-level configuration compose, with a total precedence order
14. [Interoperability and platform differences](14-interoperability-posix-windows.md) — POSIX vs. Windows divergence points and the required correction for each
15. [Type system and Deployment Profiles](15-typing-and-deployment-profiles.md) — the complete cross-language Field type mapping, and a named, reusable configuration posture distinct from a data-level Profile
16. [Relationship to existing modeling and schema-definition standards](16-relationship-to-existing-standards.md) — UML, JSON Schema, YAML, TOML
17. [Versioning](17-versioning.md) — how this standard itself evolves

Sections 10–16 were added after this document set's first draft, while its status remained
`Draft`; see [17.1](17-versioning.md#171-this-standards-own-versioning) for why that is ordinary
Draft evolution rather than an errata or a breaking change. Stub files at `10-versioning.md`,
`11-observability-and-profiling.md`, `12-configuration-precedence-and-levels.md` and
`13-execution-and-hosting-models.md` point to the files above — content landed at those numbers
first and had to move once `11-execution-models.md` claimed section 11 concurrently; the
authoring toolchain could not rename or delete the original files in place.

## Relationship to existing monorepo conventions

This standard builds on, and does not replace, the monorepo-wide engineering rules in the root
[`AGENTS-REPO.md`](../../../../AGENTS-REPO.md) — in particular its rules on SOLID/DRY/modular design,
mandatory property-based testing, fuzzing and mutation testing, UTF-8/LF text encoding, and
POSIX/Windows PowerShell process conventions. Section 9 makes the testing rules explicit
conformance requirements for any Requirements-As-Code implementation; section 10 names the
architectural patterns the SOLID/DRY/modular rules already imply in this standard's own shapes;
section 14 extends the monorepo's platform conventions into this standard's own interoperability
requirements.
