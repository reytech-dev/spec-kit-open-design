---
description: "Normalize Open Design assets into global, specification-ready design artifacts for downstream /speckit.specify executions."
---

# Open Design Process Command

## User Input

```text
$ARGUMENTS
```

Consider user input before proceeding. Supported arguments:

- `--source <path>`: override configured source directories for this run.
- `--output <path>`: override configured output directory for this run.
- `--force`: regenerate all artifacts even when source hashes are unchanged.
- `--check`: perform stale/missing validation only; do not write files. Equivalent to `/speckit.open-design.check` behavior.
- `--diff`: emphasize asset and artifact changes in the final summary.
- `--no-history`: skip writing a timestamped history snapshot for this run.
- `--no-mcp`: disable Open Design MCP retrieval for this run even when configured.
- `--mcp-project <name-or-id>`: override the configured Open Design MCP project. Use `current` to use the project currently open in Open Design.
- `--mcp-materialize <path>`: override the staging directory used for MCP-retrieved assets.
- `--mcp-required`: fail if MCP is enabled but unavailable or returns no assets.

## Purpose

Normalize Open Design assets into readable, machine-consumable artifacts before specification generation. The generated artifacts provide global project design context for later `/speckit.specify` executions.

This command is intentionally global-project-scoped. It does not require an active spec-kit feature and does not write to `specs/<feature>/`.

## Non-Goals and Safety Rules

- Do not patch `.specify/scripts/*`.
- Do not patch installed `speckit.specify`, `speckit.plan`, `speckit.tasks`, or other command files.
- Do not add `design-processing/` to spec-kit core prerequisite scripts.
- Do not mutate source design assets.
- Do not mutate the Open Design project through MCP.
- Do not invent design constraints without marking them as inferred.
- Do not silently discard ambiguity; report warnings.

## Output Contract

Default output directory:

```text
designs/design-processing/
```

Generated artifacts:

```text
manifest.json
normalized-assets.json
design-tokens.json
component-contracts.md
page-structures.md
state-variants.yaml
data-mappings.json
specify-context.md
change-summary.md
history/<timestamp>/...
```

`specify-context.md` is the compact, normative artifact that `/speckit.specify` should read and respect.

## Phase 0: Configuration and Argument Resolution

1. Resolve repository root.
2. Load configuration with this precedence:
   1. Extension defaults from `extension.yml`.
   2. Project config: `.specify/extensions/open-design/open-design-config.yml`.
   3. Local project override: `.specify/extensions/open-design/open-design-config.local.yml`, if present.
   4. Environment variables with prefix `SPECKIT_OPEN_DESIGN_`, if present.
   5. Explicit command arguments.
3. Determine source directories:
   - Use `--source` if provided.
   - Otherwise use configured `source.directories`.
   - Ignore missing source directories.
4. Determine output directory:
   - Use `--output` if provided.
   - Otherwise use configured `output.directory`.
   - Default: `designs/design-processing/`.
5. Determine history behavior:
   - Use configured `output.history` and `behavior.preserve_history`.
   - Disable history if `--no-history` is present.
6. Resolve MCP behavior:
   - Use configured `mcp.enabled`, unless `--no-mcp` is present.
   - Use configured `mcp.server`; default: `open-design`.
   - Use configured `mcp.project`; default: `current`.
   - Use `--mcp-project` if provided.
   - Use configured `mcp.materialize_sources`; default: `true`.
   - Use configured `mcp.staging_directory`; default: `designs/design-processing/.staging/open-design-mcp`.
   - Use `--mcp-materialize` if provided.
   - Treat MCP as required if `--mcp-required` is present or `mcp.fail_on_unavailable: true`.
7. Ensure output directory is outside source processing exclusions.
8. If MCP is enabled, execute Phase 0.5 before deciding whether source directories exist.
9. If no source directory exists after MCP materialization:
   - If `behavior.fail_on_no_assets: true`, stop with a clear error.
   - Otherwise write no artifacts and report that no Open Design assets were found.

## Phase 0.5: Optional Open Design MCP Source Resolution

This phase runs only when `mcp.enabled: true` and `--no-mcp` is not present.

Use the configured MCP server as an additional source of Open Design assets before normal file-system discovery.

MCP server defaults:

```yaml
mcp:
  enabled: true
  server: open-design
  project: current
  artifact_strategy: live-first
  materialize_sources: true
  staging_directory: designs/design-processing/.staging/open-design-mcp
  fail_on_unavailable: false
```

Required MCP behavior:

1. Use the configured MCP server name, default `open-design`.
2. Discover Open Design project files with `search_files`.
3. If `mcp.project` is `current`, use the currently open Open Design project.
4. Prefer assets in this order:
   - explicit Open Design metadata
   - generated artifacts
   - entry HTML
   - CSS and token files
   - JSX, TSX, JS, and TS components
   - JSON metadata
   - Markdown design notes
   - SVG and icon assets
   - raster or media references
5. Retrieve text/code assets with `get_file`.
6. Retrieve generated or renderable Open Design outputs with `get_artifact`.
7. Do not mutate the Open Design project through MCP.
8. Do not call non-Open-Design MCP tools for this phase.

When `mcp.materialize_sources: true`:

1. Create or refresh the staging directory:

   ```text
   designs/design-processing/.staging/open-design-mcp/
   ```

2. Write retrieved MCP assets into the staging directory using stable, collision-safe relative paths.
3. Preserve the original Open Design path, artifact id, or MCP resource identifier as provenance metadata.
4. Add the staging directory to the source directories for this run.
5. Process the staged files through the same normalization, hashing, and artifact-generation phases as ordinary file-system assets.

When `mcp.materialize_sources: false`:

1. Build an in-memory virtual source inventory from MCP responses.
2. Continue normalization using the same schemas and confidence rules as file-system assets.
3. Still record source provenance in `normalized-assets.json` and `manifest.json`.

MCP failure behavior:

- If MCP is unavailable and file-system sources exist, continue with a warning unless MCP is required.
- If MCP is unavailable and no file-system sources exist, fail only when `mcp.fail_on_unavailable: true`, `--mcp-required` is present, or `behavior.fail_on_no_assets: true`.
- If MCP returns no assets, apply the same no-assets policy as file-system discovery.
- All MCP failures, skipped assets, unsupported artifact types, and fallback behavior must be recorded in `change-summary.md`.

## Phase 1: Asset Discovery and Normalization

Discover all supported Open Design assets according to `source.include` and `source.exclude`.

Supported asset categories:

| Category | Extensions | Processing confidence | Treatment |
|---|---|---:|---|
| Markup | `.html`, `.htm` | High | Parse DOM, CSS, scripts, titles, visible text, routes, components, states. |
| Styles | `.css` | High | Parse tokens, rules, media queries, pseudo states, selectors. |
| Structured metadata | `.json`, `.md` | Medium/High | Parse labels, design notes, component descriptions, generated Open Design metadata. |
| Components/code | `.jsx`, `.tsx`, `.js`, `.ts` | Medium/High | Extract component structure, props, class names, variants, slots, states, and visible copy when inferable. |
| Vector | `.svg` | Medium | Extract colors, dimensions, symbols, ids, titles, accessible labels. |
| Schemas | `.graphql`, `.graphqls`, `openapi.*`, `.proto` | High | Use for data-mapping inference. |
| Raster images | `.png`, `.jpg`, `.jpeg`, `.webp` | Low/Reference | Record as visual references; do not infer exact tokens unless metadata/text is available. |
| Documents/decks | `.pdf`, `.pptx` | Medium/Reference | Extract available metadata/text if possible; otherwise record as reference assets. |
| Archives | `.zip` | Depends | Extract into temporary staging and process recursively. |
| Motion/video | `.mp4`, `.mov` | Reference | Record as motion references; document that interaction/motion details require review unless metadata is present. |

For each discovered asset, record:

```json
{
  "path": "designs/screens/login.html",
  "original_path": "login.html",
  "source": "filesystem|mcp:get_file|mcp:get_artifact",
  "kind": "html",
  "size_bytes": 12345,
  "hash": "sha256:...",
  "confidence": "high",
  "processable": true,
  "warnings": []
}
```

Write `normalized-assets.json` with:

- source mode: `filesystem`, `mcp`, or `mixed`
- source directories
- MCP server and project, if MCP was used
- discovered assets
- MCP provenance for each retrieved file or artifact
- staging extraction results
- processing confidence
- warnings
- ignored files summary

## Phase 2: Change Detection

1. Read previous `manifest.json` if present.
2. Compare current asset hashes to previous hashes.
3. Classify each asset:
   - `added`
   - `modified`
   - `removed`
   - `unchanged`
4. Determine whether regeneration is required:
   - Required if any asset is added, modified, or removed.
   - Required if any expected artifact is missing.
   - Required if MCP source inventory changed.
   - Required if `--force` is present.
5. If `--check` is present:
   - Do not write artifacts.
   - Report whether artifacts are current, stale, missing, or inconsistent.
   - Stop after reporting.
6. If no regeneration is required:
   - Report "Open Design artifacts are up to date."
   - Still show the location of `specify-context.md`.

## Phase 3: Extract Design Tokens → `design-tokens.json`

Use high-confidence structured sources first: HTML `<style>`, linked CSS files, component files, SVG attributes, JSON design metadata, and Markdown design-system notes.

Extract:

1. Palette:
   - CSS custom properties beginning with `--`.
   - Repeated hardcoded colors from CSS and SVG.
   - Semantic names from token names, class context, ids, labels, or Open Design metadata.
2. Typography:
   - font families
   - font sizes
   - font weights
   - line heights
   - letter spacing
   - text transform patterns
3. Spacing:
   - padding
   - margin
   - gap
   - grid/list spacing
4. Radius:
   - border radius values
   - pill/circle usage
5. Shadow/elevation:
   - box shadows
   - drop shadows
   - layered elevation rules
6. Breakpoints:
   - media query thresholds
   - layout changes per breakpoint
7. Motion:
   - transition durations
   - easing functions
   - animation names
   - reduced-motion notes if present
8. Asset references:
   - icons
   - logos
   - image assets
   - videos/motion artifacts

Deduplication rules:

- Same token name and same value: keep one entry and list all source files.
- Same token name and different values: keep all conflicts and emit warning.
- Same value and different semantic contexts: keep separate semantic tokens when usage differs.
- Hardcoded values repeated across multiple files become implicit tokens with generated names.

Output schema:

```json
{
  "version": 1,
  "source_mode": "filesystem|mcp|mixed",
  "source": {
    "mode": "filesystem|mcp|mixed",
    "directories": ["designs/"],
    "files": 0,
    "generated_at": "<timestamp>",
    "mcp": {
      "enabled": false,
      "server": "open-design",
      "project": "current",
      "retrieved_at": null,
      "staging_directory": null
    }
  },
  "palette": {},
  "typography": {
    "families": [],
    "scale": []
  },
  "spacing": {},
  "radius": {},
  "shadow": {},
  "breakpoints": {},
  "motion": {},
  "asset_references": [],
  "warnings": []
}
```

## Phase 4: Identify Component Contracts → `component-contracts.md`

Use the normalized asset model to detect reusable UI structures.

Detection order:

1. Explicit Open Design component metadata, if present.
2. Repeated HTML class names or DOM structures.
3. Component files from MCP or local sources.
4. CSS selectors reused across screens.
5. SVG symbol/id reuse.
6. Markdown/JSON design-system references.
7. Repeated visual asset usage.

Detection heuristics:

- Recurring class names in 2+ files indicate candidate components.
- Identical CSS rules with different class names indicate variants of the same component.
- Repeated DOM subtree shapes indicate reusable components.
- Repeated component exports, prop names, or slots indicate reusable components.
- `:hover`, `:focus`, `:active`, `.active`, `.disabled`, `[aria-*]` selectors indicate states.
- Components derived from raster/video-only references must be marked low confidence.

For each component, document:

- Component name, PascalCase.
- Derivation source and confidence level.
- Appears in source files.
- Representative DOM or artifact structure.
- CSS/design-token mapping.
- States present in designs.
- States absent but required by implementation.
- Props interface, if inferable.
- Content slots.
- Accessibility notes.
- Warnings.

## Phase 5: Map Page, Screen, Flow, and Deck Structures → `page-structures.md`

Document renderable user-facing structures.

Supported structure types:

- web page
- mobile screen
- desktop screen
- slide/deck page
- modal/dialog
- state variant screen
- HyperFrame/artifact frame
- static image reference
- video/motion reference

For each structure, capture:

1. Metadata:
   - title
   - source file
   - original Open Design path or artifact id, if MCP-sourced
   - inferred route or identifier
   - app area/domain
   - authentication requirement, if inferable
   - confidence
2. Layout type:
   - app shell
   - centered form/card
   - marketing/landing
   - dashboard/grid
   - detail view
   - list/table view
   - wizard/flow
   - deck/slide
   - media/reference
3. Component tree:
   - component references from `component-contracts.md`
   - data binding points
   - conditional visibility
   - interactive targets
4. Responsive behavior:
   - breakpoints
   - layout shifts
5. Script/interaction notes:
   - toggles
   - filtering
   - form validation
   - navigation
   - animation/motion
6. Accessibility notes:
   - headings
   - landmarks
   - labels
   - focus order, if inferable
7. Warnings.

## Phase 6: Define State Variants → `state-variants.yaml`

State variants come from two sources:

1. Explicit design variants:
   - Companion files such as `dashboard-loading.html`, `login-error.html`, `screen-empty.png`.
   - Open Design metadata naming variants, frames, states, or flows.
2. Inferred missing states:
   - Only infer states when justified by structure.
   - Mark all inferred states with `shown_in_design: false`.
   - Include the inference rule that produced the state.

Inference rules:

| Characteristic | Missing states to add |
|---|---|
| Has data list/table/grid | `loading`, `empty`, `error` |
| Has form | `submitting`, `validation-error`, `submit-error` |
| Has authentication | `unauthenticated-redirect` |
| Is auth page | `already-authenticated-redirect` |
| Has detail view | `not-found` |
| Has file upload | `uploading`, `file-too-large`, `wrong-file-type` |
| Has pagination | `single-page`, `last-page` |
| Has async media or animation | `media-loading`, `media-error`, `reduced-motion` |

## Phase 7: Generate Data Mappings → `data-mappings.json`

Use schema files discovered in Phase 1:

- GraphQL: `.graphql`, `.graphqls`
- OpenAPI: `openapi.yaml`, `openapi.yml`, `openapi.json`, or files matching `openapi.*`
- Protobuf: `.proto`
- JSON schemas: `.schema.json`, if present

For each page/screen/flow:

1. Identify data binding points from `page-structures.md`.
2. Match bindings to schema types and operations.
3. Infer likely operations only when the schema supports them.
4. Mark unmatched bindings as warnings.

If no schema files are found, still write the file with:

```json
{
  "version": 1,
  "schema_sources": [],
  "pages": {},
  "warnings": ["No schema files found; data mappings were not inferred."]
}
```

## Phase 8: Generate Specification Context → `specify-context.md`

Write a compact, normative context file for `/speckit.specify`.

This file must be concise. It is not a full duplicate of all artifacts.

Required sections:

```markdown
# Open Design Context

This project has normalized Open Design assets. New specifications MUST respect these constraints unless the user explicitly overrides them.

## Source

- Mode: `filesystem|mcp|mixed`
- MCP server: `open-design`
- MCP project: `current`
- Generated at: `<timestamp>`

## Canonical Artifacts

- Design tokens: `designs/design-processing/design-tokens.json`
- Component contracts: `designs/design-processing/component-contracts.md`
- Page structures: `designs/design-processing/page-structures.md`
- State variants: `designs/design-processing/state-variants.yaml`
- Data mappings: `designs/design-processing/data-mappings.json`

## Specification Rules

- Reuse documented components before inventing new UI components.
- Use documented design tokens for colors, typography, spacing, radius, shadow, and motion.
- Preserve documented page structures and interaction targets.
- Include all documented state variants in functional requirements.
- Treat warnings in `change-summary.md` as clarification candidates.

## Structures Available

| Structure | Type | Source | Required Notes |
|---|---|---|---|

## Changed Since Previous Run

- ...

## Warnings Requiring Human Judgment

- ...
```

## Phase 9: Version History and Change Summary

Generate `change-summary.md` with:

- timestamp
- source mode
- MCP server and project, if used
- source directories
- asset changes since previous run
- generated artifacts
- warnings
- confidence summary
- recommended next step

If history is enabled:

1. Create `history/<timestamp>/`.
2. Copy generated artifacts into the history snapshot.
3. Include the manifest and change summary in the snapshot.

Update `manifest.json` with:

```json
{
  "version": 1,
  "generated_at": "<timestamp>",
  "source_mode": "filesystem|mcp|mixed",
  "source": {
    "mode": "filesystem|mcp|mixed",
    "directories": [],
    "asset_count": 0,
    "hash_algorithm": "sha256",
    "mcp": {
      "enabled": false,
      "server": "open-design",
      "project": "current",
      "retrieved_at": null,
      "staging_directory": null
    },
    "assets": []
  },
  "artifacts": {
    "normalized_assets": "normalized-assets.json",
    "design_tokens": "design-tokens.json",
    "component_contracts": "component-contracts.md",
    "page_structures": "page-structures.md",
    "state_variants": "state-variants.yaml",
    "data_mappings": "data-mappings.json",
    "specify_context": "specify-context.md",
    "change_summary": "change-summary.md"
  },
  "changes": {
    "added": [],
    "modified": [],
    "removed": [],
    "unchanged_count": 0
  },
  "warnings": []
}
```

## Phase 10: Final Summary

Report:

```markdown
## Open Design Processing Complete

**Source mode**: `filesystem|mcp|mixed`  
**Source**: `<source directories>`  
**MCP**: `<enabled/disabled, server, project>`  
**Output**: `designs/design-processing/`  
**Specification context**: `designs/design-processing/specify-context.md`

### Artifacts Generated

| Artifact | File | Contents |
|---|---|---|
| Normalized assets | `normalized-assets.json` | N assets discovered |
| Design tokens | `design-tokens.json` | N colors, M type tokens, K spacing values |
| Component contracts | `component-contracts.md` | N components |
| Page structures | `page-structures.md` | N structures |
| State variants | `state-variants.yaml` | N explicit, M inferred |
| Data mappings | `data-mappings.json` | N mappings / no schema found |
| Specification context | `specify-context.md` | Compact context for `/speckit.specify` |
| Change summary | `change-summary.md` | Added/modified/removed assets |

### Change Summary

- Added: ...
- Modified: ...
- Removed: ...
- Unchanged: ...

### Warnings

- ...

### Next Step

Run `/speckit.specify`. The generated `specify-context.md` should be read and treated as mandatory design context unless explicitly overridden.
```

## Quality Principles

1. Derive, do not invent.
2. Prefer structured sources over raster/media references.
3. Mark confidence levels.
4. Preserve global design context across features.
5. Keep `specify-context.md` compact and normative.
6. Preserve history and change summaries.
7. Never patch spec-kit core commands or scripts.
8. Keep Open Design MCP read-only.