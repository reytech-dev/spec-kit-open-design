---
description: "Check whether normalized Open Design artifacts are missing or stale without writing files."
---

# Open Design Check Command

## User Input

```text
$ARGUMENTS
```

Supported arguments:

- `--source <path>`: override configured source directories for this run.
- `--output <path>`: override configured output directory for this run.
- `--strict`: treat warnings as check failures.
- `--no-mcp`: disable Open Design MCP validation for this run even when configured.
- `--mcp-project <name-or-id>`: override the configured Open Design MCP project. Use `current` to use the project currently open in Open Design.
- `--mcp-required`: fail if MCP is enabled but unavailable or returns no assets.

## Purpose

Validate that Open Design normalized artifacts exist and are current without writing, regenerating, or modifying any files.

This command is suitable for manual preflight checks and future CI usage.

## Safety Rules

- Do not write files.
- Do not create directories.
- Do not patch spec-kit scripts or commands.
- Do not mutate source assets.
- Do not mutate the Open Design project through MCP.

## Phase 0: Configuration and Argument Resolution

1. Resolve repository root.
2. Load configuration with this precedence:
   1. Extension defaults from `extension.yml`.
   2. Project config: `.specify/extensions/open-design/open-design-config.yml`.
   3. Local project override: `.specify/extensions/open-design/open-design-config.local.yml`, if present.
   4. Environment variables with prefix `SPECKIT_OPEN_DESIGN_`, if present.
   5. Explicit command arguments.
3. Determine source directories.
4. Determine output directory.
5. Resolve MCP behavior:
   - Use configured `mcp.enabled`, unless `--no-mcp` is present.
   - Use configured `mcp.server`; default: `open-design`.
   - Use configured `mcp.project`; default: `current`.
   - Use `--mcp-project` if provided.
   - Treat MCP as required if `--mcp-required` is present or `mcp.fail_on_unavailable: true`.

## Phase 0.5: Optional Open Design MCP Source Inventory

This phase runs only when `mcp.enabled: true` and `--no-mcp` is not present.

Because this command is check-only, it must not materialize MCP assets to disk.

Required MCP check behavior:

1. Use the configured MCP server name, default `open-design`.
2. Use `search_files` to build a virtual source inventory for the configured Open Design project.
3. If `mcp.project` is `current`, use the currently open Open Design project.
4. Prefer MCP metadata, resource identifiers, artifact identifiers, paths, sizes, modified timestamps, and hashes when available.
5. Call `get_file` only when required to compute a missing hash for comparison with `manifest.json`.
6. Call `get_artifact` only when required to validate a manifest entry that came from an artifact rather than a file.
7. Do not write files.
8. Do not create staging directories.
9. Do not mutate the Open Design project through MCP.

Virtual MCP inventory entries must be comparable with `manifest.json` entries:

```json
{
  "source": "mcp:get_file",
  "server": "open-design",
  "project": "current",
  "original_path": "index.html",
  "artifact_id": null,
  "hash": "sha256:...",
  "modified_at": null
}
```

MCP failure behavior:

- If MCP is unavailable and file-system sources exist, continue with a warning unless MCP is required.
- If MCP is unavailable and no file-system sources exist, fail only when `mcp.fail_on_unavailable: true`, `--mcp-required` is present, or `behavior.fail_on_no_assets: true`.
- If MCP returns no assets, apply the same no-assets policy as file-system discovery.
- If `--strict` is present, warnings from MCP unavailability, incomplete metadata, missing hashes, or skipped unsupported assets fail the check.

## Phase 1: Discover Source Assets

Discover assets using the configured include/exclude patterns.

If MCP is enabled and available, merge the virtual MCP inventory with file-system source assets for comparison.

Do not write MCP assets to disk during check mode.

If no assets are found:

- Pass if `behavior.fail_on_no_assets: false`.
- Fail if `behavior.fail_on_no_assets: true`.

## Phase 2: Read Manifest

Read:

```text
designs/design-processing/manifest.json
```

If missing, report:

```text
Open Design artifacts are missing. Run /speckit.open-design.process.
```

Then fail the check.

## Phase 3: Compare Hashes

1. Compute current source asset hashes.
2. Compare against `manifest.json`.
3. Classify changes:
   - added
   - modified
   - removed
   - unchanged
4. Verify required artifacts exist:
   - `normalized-assets.json`
   - `design-tokens.json`
   - `component-contracts.md`
   - `page-structures.md`
   - `state-variants.yaml`
   - `data-mappings.json`
   - `specify-context.md`
   - `change-summary.md`
5. If manifest source mode is `mcp` or `mixed`, verify that MCP provenance in the current inventory is compatible with the manifest.
6. If `--strict` is present, treat warnings as failures.

## Phase 4: Report Status

If no changes and all artifacts exist:

```markdown
## Open Design Check Passed

Artifacts are current.

**Specification context**: `designs/design-processing/specify-context.md`
```

If source assets changed:

```markdown
## Open Design Check Failed

Artifacts are stale. Run `/speckit.open-design.process`.

### Changes

- Added: ...
- Modified: ...
- Removed: ...
```

If artifacts are missing:

```markdown
## Open Design Check Failed

Required generated artifacts are missing. Run `/speckit.open-design.process`.

### Missing Artifacts

- ...
```

If warnings exist and `--strict` is present:

```markdown
## Open Design Check Failed

Warnings are present and `--strict` was used.

### Warnings

- ...
```

If MCP is configured but unavailable:

```markdown
## Open Design Check Failed

Open Design MCP is enabled but unavailable.

### Details

- MCP server: `open-design`
- Project: `current`
- Required: true|false
```

## Exit Semantics

For agent execution, express the result clearly in the final response:

- Passed: artifacts current.
- Failed: artifacts missing, stale, MCP-unavailable when required, or strict warnings present.
- Skipped: no assets found and configuration allows that.