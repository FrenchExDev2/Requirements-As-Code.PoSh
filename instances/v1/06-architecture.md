# 6. Architecture: Contracts / Runtime / Design-Time

Every conforming implementation for a language MUST ship three artifacts with this shape and
this dependency direction:

```text
Contracts  <───────────┐
   ▲                    │
   │ (depends on)       │ (depends on)
Runtime               Design-Time API
```

- **Contracts** depends on nothing from this standard. It MUST contain only the data shapes from
  [section 3](03-core-model.md) and the pure interfaces/traits needed to evaluate a Check
  (`external` callback signature, expression evaluator interface) — no I/O, no file formats, no
  configuration loading.
- **Runtime** depends only on Contracts. It MUST contain: a Catalog loader, the resolver
  (section 4), the verification algorithm (section 5), and configuration loading (section 8). It
  MUST NOT depend on Design-Time, directly or transitively — a production deployment that only
  verifies requirements MUST be able to ship Contracts+Runtime alone.
- **Design-Time API** MAY depend on Contracts and on Runtime's model/result types (to offer
  "verify this catalog right now, in my editor" authoring feedback), but exists for build-time
  and authoring-time use: typed builders for Requirement Types/Instances/Profiles, a Type/Profile
  registry client, catalog linting, and (optionally) the ahead-of-time Apply operation (6.4).
  Nothing in a running production Host MAY require Design-Time to be present.

This mirrors the monorepo's own SOLID guidance: a cohesive responsibility per module, dependents
depending on stable contracts rather than on each other's internals, and no scattering of
command-specific logic across unrelated components.

A Runtime MAY also run out-of-process, as a long-lived Service reachable over a network API
instead of linked into the Host — on a VM or inside a container. This changes nothing above:
Contracts still define the shape, Runtime still verifies, Design-Time still never ships to
production. See [section 11](11-execution-models.md) for the Service API and deployment
contract.

## 6.1 Why three artifacts and not one

A single merged library would force every production deployment to carry authoring-time
concerns (registry clients, linting, builder fluent APIs) it never uses, and would make it
impossible to update Design-Time tooling (e.g. richer IDE diagnostics) without re-shipping and
re-certifying the Runtime that is actually running in production. Splitting them is what makes
"generate a runtime and a design-time API separately" — the standard's own charter — possible.

## 6.2 The Generator

A **Generator** is the tool, conformant to this standard, that reads Requirement Types, Profiles
and Catalogs (section 3) and emits a target language's Contracts/Runtime/Design-Time artifacts.
A Generator:

- MUST be driven by the configuration schema in [section 8](08-configuration.md) for which
  Catalogs/Profiles/Types to include and which target language(s) to emit.
- MUST emit artifacts whose public surface is traceable back to section 3's model: a generated
  Design-Time type for a Requirement Type MUST expose that type's Fields with matching names and
  compatible native types (e.g. `string[]` → `List<string>` / `Vec<String>` /
  `std::vector<std::string>` / `[string[]]`).
- MAY emit either source code to be compiled by the Host's own build, or a prebuilt binary
  (DLL/`.so`/`.dylib`/crate/static lib) — the standard does not mandate one over the other.

## 6.3 Naming convention

Artifacts SHOULD be named `<Namespace>.RequirementsAsCode.Contracts`,
`<Namespace>.RequirementsAsCode.Runtime`, and `<Namespace>.RequirementsAsCode.DesignTime` (or
that ecosystem's idiomatic equivalent — see section 7 for exact per-language conventions), so
that the same three-artifact shape is recognizable across every language binding.

## 6.4 Ahead-of-time vs. just-in-time resolution

Section 4's resolution algorithm MAY run at two different times; both are conformant, and a
Runtime MUST support being given an already-resolved effective Catalog so that ahead-of-time
resolution is possible:

- **Ahead-of-time (AOT).** The Generator (via Design-Time) resolves `extends` and every Profile
  Application once, at build/generation time, and emits the flat effective Catalog as the
  Runtime artifact's embedded data (or an adjacent file the Runtime loads verbatim). The Runtime
  needs no resolver at all in this mode — smaller, faster, and unable to drift from what was
  reviewed at build time.
- **Just-in-time (JIT).** The Runtime includes the resolver and performs section 4's resolution
  itself when it loads raw Types/Profiles/Catalogs at startup — needed when Catalogs are
  hot-reloaded or come from a registry not available at build time.

A Host chooses the mode via configuration (section 8, `resolutionMode`); per 4.3's idempotency
requirement, the two modes MUST produce byte-for-byte identical effective Catalogs for the same
input, so switching modes is never an observable behavior change.
