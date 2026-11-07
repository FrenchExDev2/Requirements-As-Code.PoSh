# 16. Relationship to existing modeling and schema-definition standards

This standard deliberately reuses notation and schema languages that already exist rather than
inventing new ones, per the monorepo's own DRY rule applied at the standards level. This section
states exactly which existing standards this document set leans on, how far that reuse goes, and
where this standard explicitly stops short of depending on them.

## 16.1 UML (OMG UML 2.5.1, the latest published formal version)

Diagrams in this document set (sections 10, 12, 13, 14, 15) use UML-flavored notation — generalization
(`<|--`), composition (`*--`), sequence diagrams — because a reader already fluent in UML needs no
new notation to follow them. Concretely:

- A Requirement Type's `extends` chain ([4.2](04-application-semantics.md#42-composition-extends))
  is informally equivalent to UML **Generalization**.
- A Profile's `instances` ([3.3](03-core-model.md#33-profile)) is informally equivalent to UML
  **Composition**.
- A Check's `expression` ([3.1](03-core-model.md#31-requirement-type), 5.2) plays the role OCL
  (Object Constraint Language — part of the UML family) invariants play in a UML model: a
  constraint expressed as data, attached to a type, evaluated against an instance. This is
  precedent, external to this standard, for "constraints as data" rather than this standard's own
  invention.

This standard does **not** require UML tooling to author or verify a Catalog, and does **not**
define a UML Profile (OMG's own technical term for extending the UML metamodel itself) — a reader
MUST NOT conflate that OMG term with this standard's own **Profile** ([3.3](03-core-model.md#33-profile))
or with **Deployment Profile** ([15.5](15-typing-and-deployment-profiles.md#155-deployment-profile)).
Three distinct "profile"s exist in the surrounding vocabulary; this standard uses exactly two of
them, both defined in this document set, and borrows UML's diagram notation without borrowing
UML's own "Profile" term.

## 16.2 JSON Schema (draft 2020-12)

Already the canonical shape language for sections 3, 5, 8, 11, 12: machine-checkable, language
-neutral, and directly Generator-friendly ([6.2](06-architecture.md#62-the-generator)). It is
widely implemented across every target ecosystem this standard names — `System.Text.Json`
schema validation or `Newtonsoft.Json.Schema` in .NET, `Test-Json` in PowerShell, the
`jsonschema` crate in Rust, `valijson`/`nlohmann-json-schema-validator` in C++ — which is why it
was chosen over a bespoke IDL: no binding has to build a schema validator from scratch.

## 16.3 YAML (1.2.2)

The recommended **authoring** syntax for Catalogs, Profiles and Requirement Types (human-edited,
supports comments, already used by every example in sections 3, 4, 8, 15). A conforming YAML
document MUST deserialize to exactly the shape JSON Schema (16.2) defines for it; a parser MAY
resolve YAML-only authoring conveniences (anchors/aliases `&`/`*`, merge keys `<<:`) but the
resolved document MUST already satisfy the schema with no residual YAML-only construct — these
are serialization convenience, never a substitute for this standard's own `extends` semantics
(4.2).

A conforming YAML parser MUST use **YAML 1.2** boolean rules (`true`/`false` only) and MUST NOT
accept the legacy YAML 1.1 bare sigils `yes`/`no`/`on`/`off`/`y`/`n` as booleans. This is called
out explicitly because it is a well-documented real-world interoperability failure (informally,
"the Norway problem": a bare `NO` — Norway's ISO country code — silently parsed as boolean
`false` by a YAML 1.1 parser) and is exactly the class of silent, format-specific drift this
standard exists to eliminate at the data-modeling level, not reintroduce at the authoring-syntax
level.

## 16.4 TOML (1.0.0)

Acceptable, and often preferable, for flat Configuration documents
([section 8](08-configuration.md), [section 15](15-typing-and-deployment-profiles.md#155-deployment-profile)'s
Deployment Profiles) where nesting stays shallow. **Not recommended** for Catalogs/Profiles with
deep Requirement Instance nesting: TOML's array-of-tables syntax (`[[instances]]`) degrades in
readability once `values`/`subjectSelector` nest further, which is precisely the data this
standard's Catalogs carry. A binding MAY still accept TOML for a Catalog if a Host explicitly
asks for it, but MUST NOT emit TOML as its own default scaffold/suggestion for authoring a
Catalog or Profile.

## 16.5 Format summary

| Format | Permitted for | Recommended for | Not recommended for |
|---|---|---|---|
| JSON | Everything (sections 3, 5, 8, 11, 12) | Machine-generated output (Verification Reports, 5.3) | Hand-authoring (no comments) |
| YAML | Catalogs, Profiles, Requirement Types, Configuration | Hand-authored Catalogs/Profiles (3, 15.5) | — |
| TOML | Configuration, Deployment Profiles | Flat Configuration documents (8, 15.5) | Deeply nested Catalogs/Profiles |

## 16.6 Why this section exists

Sections 10–16 (this one included) added substantial new normative material to a still-**Draft**
instance; see [17.1](17-versioning.md#171-this-standards-own-versioning) for why that is ordinary
Draft evolution rather than an errata or a breaking change. Naming, explicitly, which parts of
that material are this standard's own invention (the data model, sections 3–5; the algorithm,
section 5; the pattern catalogue, section 10) versus which
parts are deliberate reuse of an existing standard (this section) is itself a DRY/SOLID
discipline applied to the standard's own text: it prevents a future reader from mistakenly
treating UML, JSON Schema, YAML or TOML semantics as if this standard had silently redefined them.
