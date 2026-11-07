# 17. Versioning

## 17.1 This standard's own versioning

This standard is versioned independently of any implementation. A published version is
immutable once its status moves from `Draft` to `Final`: text under `instances/v1/` MUST NOT
change its normative meaning after that point.

- **Errata** (clarifications that do not change conformance outcomes — typos, examples,
  non-normative explanation) MAY be applied in place to `instances/v1/`, recorded in a
  `CHANGELOG.md` added to that directory once the first errata lands.
- **Breaking changes** (anything that would change a Verification Report for previously-valid
  input, or that changes a schema in section 3, 5 or 8 in an incompatible way) MUST be published
  as a new `instances/v2/`, alongside `v1/` which remains available and unchanged.
- Sections [10](10-design-patterns-and-strategies.md)–[16](16-relationship-to-existing-standards.md)
  were added to this document set while its status was still `Draft`. Adding wholly new,
  additive sections — none of which changes a schema already defined in sections 3, 5 or 8, or
  changes the outcome of the algorithm in section 5, for previously-valid input — is ordinary
  Draft evolution under this same rule, not an errata and not a breaking change; it would only
  need a new `instances/v2/` if it changed an existing conformance outcome, which it does not.

## 17.2 Requirement Types and Profiles versioning

Requirement Types and Profiles carry their own `version` (semver, section 3). A Requirement
Instance's `type` ref and a Profile Application's `profile` ref MUST pin an exact version
(`id@version`), never a range — reproducibility of a Verification Report for a given Catalog is
a correctness property, not a convenience.

- A new, backward-compatible Field or Check on a Requirement Type (or a new Instance on a
  Profile) is a `minor` bump.
- Removing or changing the meaning of an existing Field, Check, or Parameter is a `major` bump;
  every existing Instance/Application pinned to the old version keeps working unchanged against
  that old version.
- Non-semantic edits (description/title wording) are a `patch` bump.

## 17.3 Deprecation

A Requirement Type, Profile, or Check MAY be marked `deprecated` in its `metadata`. A Catalog
resolving a `deprecated` reference MUST still produce correct Bindings and Results (it is not an
error), but a Runtime SHOULD surface a distinct deprecation notice in the Verification Report so
authors can migrate before the next major version retires it.

## 17.4 Status of this document

This is `instances/v1`, status **Draft**. It becomes **Final** once at least one language
binding reaches Conformance Level L1 against it and the conformance test vectors referenced in
[7.1](07-language-bindings.md#71-conformance-test-vectors) exist and pass.
