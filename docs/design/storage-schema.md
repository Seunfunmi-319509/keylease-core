# Storage Schema Design

## Overview

This design introduces schema versioning to the KeyLease Registry contract storage. Every deployment records a schema version in instance storage, and state-changing entry points verify the version before proceeding. This allows the contract to evolve its storage layout safely across Wasm upgrades.

## Schema Versioning

### `STORAGE_SCHEMA_VERSION`

- Current version: `1`
- Deployments before schema versioning have no stored version, treated as schema `0`
- Must be migrated to before other state mutations when the stored version differs

### `LEGACY_SCHEMA_VERSION`

- Value: `0`
- Represents deployments created before schema versioning existed
- These deployments must call `migrate(1)` after upgrading

### Schema Check Logic

Every state-changing entry point calls `require_schema(env)` which:

1. If the registry is not initialized → allow (initialization still possible)
2. If stored version == `STORAGE_SCHEMA_VERSION` → allow
3. If stored version < `STORAGE_SCHEMA_VERSION` → return `Error::SchemaMigrationRequired`
4. If stored version > `STORAGE_SCHEMA_VERSION` → return `Error::SchemaVersionUnsupported`

### `migrate(to_version)` Entry Point

- Admin only
- Can only move forward to `STORAGE_SCHEMA_VERSION` (version 1)
- Refuses downgrades to prevent older code from misreading newer data
- Records a `SchemaMigrated` event
- Repeated calls are a no-op, not an error

### Error Codes

- 20: `SchemaMigrationRequired` — stored schema is older than this Wasm expects
- 21: `SchemaVersionUnsupported` — stored schema is newer, or migration targeted an unsupported version

## Migration Path

1. Admin upgrades Wasm to new version
2. Admin calls `migrate(1)` to record the schema version
3. All subsequent state mutations proceed normally

Until migration, the registry accepts `init`, `set_token`, and `upgrade` (which is the only mutation allowed before migration per the design). All other entry points require the schema to match.