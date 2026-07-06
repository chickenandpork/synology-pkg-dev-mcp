# Synology Deployment MCP (MVP Specification)

## Purpose

A stateless, Go-based, Bazel-built MCP server that manages one or more Synology DSM systems using DSM APIs as the primary integration mechanism.

The service acts as a translation layer between LLM agents and Synology APIs, exposing deployment-oriented tools and structured data optimized for machine reasoning.

## Design Principles

### Stateless

The MCP stores no durable operational state.

All authoritative state resides in:

- Synology DSM APIs
- Package Center
- Log Center
- Container Manager
- Uploaded package staging area (on Synology)

The MCP may maintain short-lived in-memory caches for performance.

### API First

Operation order:

1. DSM API
2. Synology-supported SDK/API
3. CLI fallback (explicitly marked)
4. SSH never required

Every response includes provenance:

```json
{
  "source": {
    "type": "dsm_api",
    "api": "SYNO.Core.Package"
  }
}
```

### Agent-Oriented

The MCP should expose:

- structured JSON
- summaries
- classifications
- deployment reports

rather than raw DSM responses.

## Architecture

```text
Agent
  |
  v
MCP Server (Go)
  |
  +-- DSM Client
  +-- Package Provider
  +-- Validation Provider
  +-- Log Provider
  +-- Capability Provider
  |
  v
Synology DSM APIs
```

## Build Requirements

### Language

Go

### Build

Bazel 8+ with bzlmod

No WORKSPACE dependency.

### Runtime

Single static binary.

### Packaging

- Linux AMD64
- Linux ARM64

Container image optional.

## Core Tools

### server_info

Returns:
- version
- build metadata
- supported capabilities
- target DSM version

### get_capabilities

Lists available capability groups.

Example:
- package_management
- logging
- validation
- container_manager

### probe_capability

Performs a non-destructive capability probe.

Examples:
- Package API reachable
- Log API reachable
- Container Manager reachable

### probe_all_capabilities

Runs all probes.

Returns cached results unless force=true.

## Capability Cache

Capability probes are cached.

Default TTL:

6 hours

Example:

```json
{
  "capability": "package_management",
  "status": "ok",
  "cached": true,
  "last_checked": "..."
}
```

## Package Lifecycle

### upload_spk

Uploads an SPK to Synology-managed staging.

### begin_upload

Creates upload session.

### upload_chunk

Uploads chunk.

### complete_upload

Finalizes upload.

### list_staged_packages

Lists uploaded packages.

### inspect_spk

Extracts:
- package name
- version
- architecture
- publisher
- checksum

## Wizard Support

### inspect_wizard

Discovers package installation fields.

Returns typed schema.

Supported field types:
- string
- integer
- boolean
- enum
- secret

### validate_wizard

Validates wizard values prior to install.

### install_spk

Inputs:
- package identifier
- wizard values

### upgrade_spk

Inputs:
- package identifier
- wizard values

Existing configuration is merged with supplied values.

## Configuration Handling

### get_configuration

Returns deployed configuration.

Secret values are redacted.

Example:

```json
{
  "api_token": "***REDACTED***"
}
```

## Validation Framework

### run_validation

Runs deployment validation profile.

Supported checks:
- API reachable
- service running
- TCP port open
- HTTP endpoint healthy
- package state valid

Result:

PASS or FAIL

## Logging

### get_logs

Returns structured logs.

Schema:

```json
{
  "timestamp": "...",
  "level": "ERROR",
  "component": "installer",
  "message": "..."
}
```

### get_deployment_report

Returns:
- timeline
- errors
- warnings
- summary
- structured log excerpts

## Provenance

Every response includes:

```json
{
  "source": {
    "type": "dsm_api"
  }
}
```

or

```json
{
  "source": {
    "type": "cli_fallback"
  }
}
```

## Security

### Allowed Operations

Only explicitly implemented operations.

No:
- shell execution
- arbitrary file access
- arbitrary DSM API execution

### Secrets

Secrets:
- accepted
- transmitted
- redacted on retrieval

Never exposed in logs.

## Synology Compatibility

Target baseline:
- DSM 7.2+
- DSM 7.4+

API capabilities detected dynamically.

## Acceptance Criteria

1. Discover DSM capabilities.
2. Probe capabilities without side effects.
3. Upload SPK packages.
4. Inspect package metadata.
5. Discover wizard fields.
6. Validate wizard inputs.
7. Install packages.
8. Upgrade packages.
9. Retrieve package status.
10. Run validation profiles.
11. Retrieve structured logs.
12. Generate deployment reports.
13. Operate without SSH.
14. Operate without local persistence.
15. Prefer DSM APIs for all supported operations.