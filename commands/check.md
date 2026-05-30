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

## Purpose

Validate that Open Design normalized artifacts exist and are current without writing, regenerating, or modifying any files.

This command is suitable for manual preflight checks and future CI usage.

## Safety Rules

- Do not write files.
- Do not create directories.
- Do not patch spec-kit scripts or commands.
- Do not mutate source assets.

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

## Phase 1: Discover Source Assets

Discover assets using the configured include/exclude patterns.

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

## Exit Semantics

For agent execution, express the result clearly in the final response:

- Passed: artifacts current.
- Failed: artifacts missing, stale, or strict warnings present.
- Skipped: no assets found and configuration allows that.
