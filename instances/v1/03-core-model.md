# 3. Core model

This section defines the four data shapes every conforming implementation MUST be able to read:
**Requirement Type**, **Requirement Instance**, **Profile**, **Catalog**. Shapes are given as
JSON Schema (draft 2020-12) for precision; the authoring format MAY be YAML or JSON — both
encode the same shape.

Schema `$id`s use the `urn:frenchexdev:rac:v1:...` namespace; they are stable identifiers, not
network locations, and MUST NOT be dereferenced over the network by a conforming implementation.

## 3.1 Requirement Type

```json
{
  "$id": "urn:frenchexdev:rac:v1:requirement-type",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["id", "version", "fields", "checks"],
  "properties": {
    "id": { "type": "string", "pattern": "^[A-Za-z0-9][A-Za-z0-9._-]*$" },
    "version": { "type": "string", "pattern": "^\\d+\\.\\d+\\.\\d+$" },
    "extends": {
      "type": "array",
      "items": { "type": "string", "description": "Requirement Type ref, 'id@version'" },
      "default": []
    },
    "title": { "type": "string" },
    "description": { "type": "string" },
    "defaultSeverity": { "$ref": "urn:frenchexdev:rac:v1:severity" },
    "fields": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["name", "type"],
        "properties": {
          "name": { "type": "string" },
          "type": { "enum": ["string", "integer", "number", "boolean", "string[]", "object"] },
          "required": { "type": "boolean", "default": false },
          "default": {},
          "constraints": { "type": "object" }
        }
      }
    },
    "checks": {
      "type": "array",
      "items": { "$ref": "urn:frenchexdev:rac:v1:check" }
    },
    "metadata": { "type": "object" }
  }
}
```

```json
{
  "$id": "urn:frenchexdev:rac:v1:check",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["id", "kind", "message"],
  "properties": {
    "id": { "type": "string" },
    "kind": { "enum": ["expression", "regex", "schema", "external"] },
    "expression": { "type": "string" },
    "message": { "type": "string" },
    "severity": { "$ref": "urn:frenchexdev:rac:v1:severity" },
    "suppressible": { "type": "boolean", "default": true }
  }
}
```

```json
{
  "$id": "urn:frenchexdev:rac:v1:severity",
  "enum": ["info", "warning", "error", "blocker"]
}
```

### Example

```yaml
id: no-sync-over-async
version: 1.0.0
title: "No blocking sync call over an async API"
defaultSeverity: error
fields:
  - name: forbiddenMembers
    type: string[]
    required: true
checks:
  - id: forbidden-member-not-called
    kind: expression
    expression: "not callsAny(subject, fields.forbiddenMembers)"
    message: "Subject calls a forbidden blocking member: {{matchedMember}}"
```

## 3.2 Requirement Instance

```json
{
  "$id": "urn:frenchexdev:rac:v1:requirement-instance",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["id", "type", "subjectSelector"],
  "properties": {
    "id": { "type": "string" },
    "type": { "type": "string", "description": "Requirement Type ref, 'id@version'" },
    "status": { "enum": ["draft", "approved", "deprecated", "retired"], "default": "draft" },
    "values": { "type": "object" },
    "checkOverrides": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["checkId"],
        "properties": {
          "checkId": { "type": "string" },
          "severity": { "$ref": "urn:frenchexdev:rac:v1:severity" },
          "suppressed": { "type": "boolean", "default": false },
          "suppressionReason": { "type": "string" }
        }
      },
      "default": []
    },
    "subjectSelector": { "type": "object" },
    "metadata": { "type": "object" }
  }
}
```

A `checkOverrides` entry that sets `suppressed: true` without a `suppressionReason` is invalid
for any Check whose `suppressible` is `true` and `severity` is `error` or `blocker`: an
implementation MUST reject it (see [4.4](04-application-semantics.md#44-suppression)).

### Example

```yaml
id: aiwebsitedetector-no-sync-over-async
type: no-sync-over-async@1.0.0
status: approved
values:
  forbiddenMembers: ["Task.Result", "Task.Wait", ".GetAwaiter().GetResult()"]
subjectSelector:
  kind: glob
  pattern: "Net/FrenchExDev/AIWebsiteDetector/**/*.cs"
```

That glob is illustrative data for a Subject Selector, not an instruction to scan any real path.

## 3.3 Profile

```json
{
  "$id": "urn:frenchexdev:rac:v1:profile",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["id", "version"],
  "properties": {
    "id": { "type": "string" },
    "version": { "type": "string", "pattern": "^\\d+\\.\\d+\\.\\d+$" },
    "extends": { "type": "array", "items": { "type": "string" }, "default": [] },
    "title": { "type": "string" },
    "description": { "type": "string" },
    "parameters": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["name"],
        "properties": {
          "name": { "type": "string" },
          "type": { "enum": ["string", "integer", "number", "boolean", "string[]"] },
          "default": {},
          "required": { "type": "boolean", "default": false }
        }
      },
      "default": []
    },
    "instances": {
      "type": "array",
      "items": { "$ref": "urn:frenchexdev:rac:v1:requirement-instance" },
      "description": "May contain '${parameterName}' placeholders in values/subjectSelector."
    }
  }
}
```

### Example

```yaml
id: solid-dotnet-library
version: 1.0.0
parameters:
  - name: sourceGlob
    type: string
    required: true
instances:
  - id: no-sync-over-async-binding
    type: no-sync-over-async@1.0.0
    status: approved
    values:
      forbiddenMembers: ["Task.Result", "Task.Wait"]
    subjectSelector:
      kind: glob
      pattern: "${sourceGlob}"
```

## 3.4 Catalog

```json
{
  "$id": "urn:frenchexdev:rac:v1:catalog",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["id", "version"],
  "properties": {
    "id": { "type": "string" },
    "version": { "type": "string", "pattern": "^\\d+\\.\\d+\\.\\d+$" },
    "requirementTypes": { "type": "array", "items": { "$ref": "urn:frenchexdev:rac:v1:requirement-type" }, "default": [] },
    "profiles": { "type": "array", "items": { "$ref": "urn:frenchexdev:rac:v1:profile" }, "default": [] },
    "applications": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["profile"],
        "properties": {
          "profile": { "type": "string", "description": "Profile ref, 'id@version'" },
          "parameterValues": { "type": "object" },
          "scopeOverride": { "type": "object" }
        }
      },
      "default": []
    },
    "instances": {
      "type": "array",
      "items": { "$ref": "urn:frenchexdev:rac:v1:requirement-instance" },
      "default": [],
      "description": "Standalone instances not introduced through a profile application."
    }
  }
}
```

A Catalog is authoring input (`requirementTypes`/`profiles`/`applications`/`instances` as
written by a human) until it is **resolved** (section 4) into the **effective Catalog**: the
flat list of effective Requirement Instances a Verifier actually binds and evaluates. An
implementation MAY persist the effective Catalog (ahead-of-time resolution, see
[6.4](06-architecture.md#64-ahead-of-time-vs-just-in-time-resolution)) or recompute it at load
time (just-in-time resolution).
