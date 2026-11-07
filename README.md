# Requirements-As-Code

Requirements-As-Code (RaC) is a FrenchExDev standard for authoring, versioning, generating
runtime verifiers for, and checking software and process requirements as structured,
machine-readable data instead of free-text documents.

## What this repository is

This repository holds the **normative text** of the standard, not an implementation of it.
Implementations (Contracts/Runtime/Design-Time artifacts, generators) are expected to live in
the consuming ecosystems — this monorepo's `Net/FrenchExDev/` and `PoSh/FrenchExDev/`, and
future Rust/C++ ecosystems — and to declare which instance of this standard, and which
conformance level, they implement.

## Versioning

The standard is versioned like any other artifact: each published, immutable version lives
under `instances/vN/`. `instances/v1` is the first, currently **Draft**, version. See
[`instances/v1/10-versioning.md`](instances/v1/10-versioning.md) for the compatibility and
deprecation policy, and for when a change belongs in `v1` (errata) versus a new `v2` (breaking
revision).

## Start here

Read [`instances/v1/README.md`](instances/v1/README.md).

## Git

This directory is its own Git repository, separate from the FrenchExDev_i2 monorepo. No commit,
push or other Git operation was performed here while writing this standard.

## Once an implementation exists

Add an `AGENTS.md` here documenting the project map, dependencies, CI path filters and publish
triggers for the `Contracts`/`Runtime`/`DesignTime` packages that implement this standard, per
the monorepo convention illustrated by QuietCli2's
[`AGENTS.md`](../../PoSh/FrenchExDev/QuietCliEcosystem/FrenchExDev2.QuietCli.PoSh/AGENTS.md).
