# 7. Language bindings

This section specifies, per language, how the abstract model (sections 3–6) maps onto that
language's ecosystem. Every binding MUST produce the same Verification Report (section 5) for
the same Catalog; differences below are packaging and idiom, never semantics.

## 7.1 Conformance test vectors

A binding MUST pass the standard's conformance test vectors (fixed Catalogs with known-correct
Verification Reports, maintained alongside this standard) before it may claim any Conformance
Level ([section 9](09-conformance-and-testing.md)). This is what makes "the same catalog, every
language, same answer" a verified property rather than an aspiration.

## 7.2 .NET (C#)

- Target the SDK pinned by [`Net/FrenchExDev/global.json`](../../../../Net/FrenchExDev/global.json)
  and this monorepo's Central Package Management
  ([`Directory.Packages.props`](../../../../Net/FrenchExDev/Directory.Packages.props)) — never
  pin a package version directly in a `.csproj`.
- Package names: `FrenchExDev.RequirementsAsCode.Contracts`, `...Runtime`, `...DesignTime`,
  each its own `.slnx`-tracked project under `Net/FrenchExDev/`, per the monorepo's one-package
  one-project convention.
- Every Runtime/Design-Time entry point that performs I/O MUST be `async` and MUST return
  `Task`/`Task<T>` (or `ValueTask`/`ValueTask<T>` where allocation matters); a synchronous
  wrapper MAY be offered additionally, never instead, per
  [1.4](01-overview.md#14-design-principles).
- `external` Checks (5.2) are ordinary delegates (`Func<Subject, CheckContext, Task<CheckOutcome>>`).
- Documentation MUST follow this monorepo's convention: XML doc comments on every public class,
  interface, method and parameter, in English.

## 7.3 PowerShell

- Target PowerShell Core 7.6+, per the monorepo's `AGENTS.md`.
- The PowerShell binding is a **facade module**, not an independent implementation: it hosts the
  .NET Contracts/Runtime/DesignTime assemblies above through this monorepo's existing QuietCli2
  hosting conventions (`Auto` CLR selection between the integrated net10.0 host and the pinned
  net11.0 host — see
  [QuietCli2's `AGENTS.md`](../../../../PoSh/FrenchExDev/QuietCliEcosystem/FrenchExDev2.QuietCli.PoSh/AGENTS.md)).
  This avoids a second, divergent implementation of sections 3–5 in PowerShell.
- Exposed cmdlets SHOULD follow PowerShell's verb-noun convention against the
  `RequirementsAsCode` noun family, e.g. `Invoke-RequirementsAsCodeVerification`,
  `Get-RequirementsAsCodeCatalog`, `New-RequirementsAsCodeProfileApplication` — each a thin
  wrapper that calls into the hosted .NET Runtime/Design-Time and returns native PowerShell
  objects (not raw .NET types left for the caller to unwrap).
- Every native/engine call the module makes MUST go through the `quiet` alias (QuietCli2 Core),
  per the monorepo's `AGENTS.md`; the module itself MUST NOT shell out directly. A PowerShell
  Service Client ([11.3](11-execution-models.md#113-service-clients)) talking to a remote
  Service uses the hosted .NET `HttpClient` — that is in-process network I/O, not native CLI
  invocation, so the `quiet` requirement does not apply to it.
- Comment-based help MUST be provided for every exported function, in English, per the
  monorepo's documentation convention.

## 7.4 Rust

- Target the latest stable edition at the time a binding is built (no fixed edition pin in this
  standard; a binding's own `Cargo.toml`/`rust-toolchain.toml` records what it targets).
- Crate names: `requirements_as_code_contracts`, `requirements_as_code_runtime`,
  `requirements_as_code_designtime`, following crates.io naming norms (snake_case, no vendor
  prefix duplication inside the crate name since the registry namespace already scopes it).
- Runtime/Design-Time async entry points MUST be `async fn` returning a `Result<T, RacError>`
  (never panic across the Contracts boundary for a data-driven failure — a malformed Catalog is
  an `Err`, not a panic) on an executor-agnostic basis (no hard dependency on a specific async
  runtime baked into Contracts; a Runtime crate MAY depend on one, e.g. `tokio`, naming it
  explicitly in its own manifest).
- `external` Checks are `Fn(&Subject, &CheckContext) -> Pin<Box<dyn Future<Output = CheckOutcome> + Send>>`
  (or the then-current idiomatic async-trait-object shape).

## 7.5 C++ (latest standard)

- Target the latest published ISO C++ standard supported by the toolchains this monorepo already
  builds with (treat "latest" as a moving target resolved at implementation time, not pinned by
  this document).
- Prefer C++20/23 modules (`export module requirements_as_code.contracts;` etc.) over header-only
  distribution where the toolchain supports it; a header-only fallback MAY be offered for
  toolchains without stable modules support.
- Namespaces: `frenchexdev::requirements_as_code::contracts`, `::runtime`, `::design_time`.
- Runtime/Design-Time async entry points MUST return a future-like type (`std::future<T>`, a
  coroutine returning a project-chosen `task<T>`, or an equivalent) rather than blocking the
  calling thread; a synchronous convenience wrapper MAY additionally be offered.
- `external` Checks are `std::function<Outcome(const Subject&, const CheckContext&)>` or an
  awaitable equivalent.
- Every public class, function and parameter MUST be documented (Doxygen-style comments), in
  English, per the monorepo's documentation convention.

## 7.6 Cross-cutting requirement

No binding MAY introduce a field, Check `kind`, or Report field not defined in sections 3 and 5.
A binding needing an ecosystem-specific extension MUST propose it as a change to this standard
(section 10) rather than diverge silently — divergence here is exactly the drift RaC exists to
eliminate.
