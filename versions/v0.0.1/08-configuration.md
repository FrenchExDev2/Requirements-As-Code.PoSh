# 8. Configuration

Every Host — regardless of language — MUST be driven by the same configuration shape, so that
switching which Catalogs/Profiles apply, or switching severity threshold, or switching from a
PowerShell host to a Rust host, is a configuration change, never a code change
([1.4](01-overview.md#14-design-principles)).

## 8.1 Schema

```json
{
  "$id": "urn:frenchexdev:rac:v1:configuration",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["catalogSources"],
  "properties": {
    "catalogSources": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["kind", "location"],
        "properties": {
          "kind": { "enum": ["file", "directory", "http", "custom"] },
          "location": { "type": "string" }
        }
      },
      "minItems": 1
    },
    "typeRegistries": {
      "type": "array",
      "items": { "type": "object", "required": ["kind", "location"] },
      "default": []
    },
    "resolutionMode": { "enum": ["ahead-of-time", "just-in-time"], "default": "just-in-time" },
    "severityThreshold": { "$ref": "urn:frenchexdev:rac:v1:severity", "default": "error" },
    "shortCircuit": { "enum": ["fail-fast", "collect-all"], "default": "collect-all" },
    "report": {
      "type": "object",
      "properties": {
        "format": { "enum": ["json", "yaml"], "default": "json" },
        "path": { "type": "string" }
      }
    },
    "triggers": {
      "type": "array",
      "items": { "enum": ["startup", "ci", "scheduled", "on-demand"] },
      "default": ["on-demand"]
    },
    "scheduledCron": {
      "type": "string",
      "description": "5-field cron, required when 'scheduled' is in triggers."
    },
    "executionModel": {
      "enum": ["embedded", "service"],
      "default": "embedded",
      "description": "See section 11. 'embedded' requires no network dependency."
    },
    "serviceEndpoint": {
      "type": "string",
      "description": "Client side, when executionModel is 'service': base URL of a running Verification Service, e.g. https://rac.internal:8743."
    },
    "bindAddress": {
      "type": "string",
      "default": "0.0.0.0",
      "description": "Server side, when running as the Service: interface to listen on."
    },
    "bindPort": {
      "type": "integer",
      "default": 8743,
      "description": "Server side, when running as the Service."
    },
    "shutdownGraceSeconds": {
      "type": "integer",
      "default": 30,
      "description": "Server side: time to let in-flight verifications finish after a stop signal."
    },
    "tls": {
      "type": "object",
      "properties": {
        "certPath": { "type": "string" },
        "keyPath": { "type": "string" }
      },
      "description": "Server side: required once the Service is reachable over anything but loopback."
    },
    "profilingMode": {
      "enum": ["off", "summary", "per-binding"],
      "default": "off",
      "description": "See section 12. MUST NOT change the Verification Report; controls diagnostics volume only."
    }
  }
}
```

`executionModel` and the fields below it apply only to the `service` execution model and its
VM/container deployment profiles; see [section 11](11-execution-models.md) for their full
semantics. A Host left at the default, `embedded`, ignores them entirely. `profilingMode` is
independent of `executionModel` and applies to every hosting shape; see
[section 12](12-observability-and-profiling.md#126-profiling-must-not-change-outcomes).

## 8.2 File and discovery convention

- Default file name: `requirements-as-code.config.yaml`, discovered at the Host's working
  directory or an explicit path given to it.
- Override via a single environment variable, `RAC_CONFIG`, naming an absolute path — every
  binding MUST honor this variable identically; it is the one escape hatch that does not require
  a language-specific flag.
- A binding MAY additionally accept its own native configuration source (an `IConfiguration`
  provider in .NET, a PowerShell parameter, a `clap` flag in Rust, a CLI flag in C++) **layered
  on top of**, never instead of, the file above — the file remains the portable source of truth.
- Individual scalar fields above MAY also be overridden by a dedicated environment variable
  (`RAC_SEVERITY_THRESHOLD`, `RAC_SHORT_CIRCUIT`, `RAC_RESOLUTION_MODE`, `RAC_PROFILING_MODE`),
  distinct from `RAC_CONFIG` above (which locates the file; it does not override a value inside
  it). See [section 13](13-configuration-precedence-and-levels.md) for the complete, total
  precedence order across the file, these variables, and a Host's own native configuration.

## 8.3 Example

```yaml
catalogSources:
  - kind: file
    location: "./requirements/catalog.yaml"
resolutionMode: just-in-time
severityThreshold: error
shortCircuit: collect-all
report:
  format: json
  path: "./out/rac-report.json"
triggers:
  - ci
  - on-demand
```

The same file, unchanged, MUST produce the same Verification Report whether the Host that reads
it is the PowerShell module, the C# Runtime, the Rust Runtime, or the C++ Runtime
([7.1](07-language-bindings.md#71-conformance-test-vectors)'s conformance test vectors verify
exactly this property).
