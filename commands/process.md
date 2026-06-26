---
description: "Normalize Open Design assets into global, specification-ready design artifacts for downstream /speckit.specify executions."
---

# Open Design Process Command

## User Input

```text
$ARGUMENTS
```

Consider user input before proceeding. Supported arguments:

- `--project <slug>`: required for workbench design-context mode. Resolves paths to `workspace/design-context/<slug>/`.
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
- `--visual`: enable visual artifact ingestion. Default is `true` when `--project` is provided.
- `--no-visual`: skip visual artifact ingestion.
- `--require-visual-artifacts`: fail if design-ir.json, prototype-map.json, source-map.json, or screenshots are missing or invalid.
- `--force-route-map`: regenerate visual-regression/fixtures/route-map.json even if it already exists.

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
- Do not generate `visual-regression/package.json`, `capture-open-design.mjs`, `open-design.visual.spec.ts`, or `playwright.config.ts`.
- Do not run Playwright, install npm packages, start an HTTP server, or capture screenshots.

## Output Contract

Default output directory (when no `--project` is given):

```text
designs/design-processing/
```

When `--project <slug>` is given, defaults resolve to:

```text
workspace/design-context/<slug>/design-processing/
workspace/design-context/<slug>/visual-regression/
workspace/design-context/<slug>/handoff/
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
frontend-implementation-brief.md
change-summary.md
history/<timestamp>/...
../visual-regression/fixtures/route-map.json
../handoff/README.md
```

Consumed visual artifacts (produced by opencode-environment static crawler):

```text
design-processing/design-ir.json
../visual-regression/fixtures/prototype-map.json
../visual-regression/fixtures/source-map.json
../visual-regression/screenshots/*.png
```

`specify-context.md` is the compact, normative artifact that `/speckit.specify` should read and respect.
`design-ir.json` is consumed as the canonical rendered visual implementation contract produced by the crawler.
`frontend-implementation-brief.md` tells future frontend agents how to consume the design context.

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
4. Determine project slug:
   - Use `--project <slug>` if provided.
   - Otherwise use configured `project.slug`.
   - Default: `null` (no project-scoped path inference).
5. Determine output directory:
   - Use `--output <path>` if provided.
   - Else if `project.slug` is set: `workspace/design-context/<slug>/design-processing`.
   - Otherwise use configured `output.directory`.
   - Default: `designs/design-processing/`.
6. Determine visual processing mode and artifact paths:
   - Resolve whether visual artifact ingestion is enabled:
     - `--visual` → enabled.
     - `--no-visual` → disabled.
     - `--project <slug>` provided without `--no-visual` → enabled.
     - Otherwise use configured `visual.enabled` (default: `true`).
   - Resolve whether visual artifacts are required:
     - `--require-visual-artifacts` → required.
     - Otherwise use configured `visual.required` (default: `false`).
   - Resolve whether to force route-map regeneration:
     - `--force-route-map` → regenerate.
     - Otherwise use configured behavior (default: `false`, preserve existing route-map.json).
   - Determine canonical preview URL:
     - Use configured `visual.canonical_url_template`.
     - Default: `http://design-preview:80/design-context/<project-slug>/index.html`.
   - Determine artifact paths (defaults only; override with configured `visual.artifacts` if set):
     - `design_context_root`: `workspace/design-context/<project-slug>/`
     - `design_processing_output`: `workspace/design-context/<project-slug>/design-processing/`
     - `visual_root`: `workspace/design-context/<project-slug>/visual-regression/`
     - `design_ir`: `workspace/design-context/<project-slug>/design-processing/design-ir.json`
     - `prototype_map`: `workspace/design-context/<project-slug>/visual-regression/fixtures/prototype-map.json`
     - `source_map`: `workspace/design-context/<project-slug>/visual-regression/fixtures/source-map.json`
     - `screenshots_directory`: `workspace/design-context/<project-slug>/visual-regression/screenshots/`
     - `route_map`: `workspace/design-context/<project-slug>/visual-regression/fixtures/route-map.json`
     - `handoff_output`: `workspace/design-context/<project-slug>/handoff/`
7. Determine history behavior:
   - Use configured `output.history` and `behavior.preserve_history`.
   - Disable history if `--no-history` is present.
8. Resolve MCP behavior:
   - Use configured `mcp.enabled`, unless `--no-mcp` is present.
   - Use configured `mcp.server`; default: `open-design`.
   - Use configured `mcp.project`; default: `current`.
   - Use `--mcp-project` if provided.
   - Use configured `mcp.materialize_sources`; default: `true`.
   - Use configured `mcp.staging_directory`; default: `designs/design-processing/.staging/open-design-mcp`.
   - Use `--mcp-materialize` if provided.
   - Treat MCP as required if `--mcp-required` is present or `mcp.fail_on_unavailable: true`.
9. Ensure output directory is outside source processing exclusions.
10. If a project slug is set, add `workspace/design-context/<slug>/source/open-design-export` to the source directory list for this run.
11. If MCP is enabled, execute Phase 0.5 before deciding whether source directories exist.
12. If no source directory exists after MCP materialization:
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

## Phase 3: Ingest Workbench Visual Artifacts

This phase runs when visual artifact ingestion is enabled (`--visual` or `visual.enabled: true`).

It validates and ingests visual artifacts produced by the opencode-environment static crawler. The extension does NOT run Playwright, does NOT generate crawler code, does NOT start an HTTP server, and does NOT capture screenshots.

The crawler command that produces these artifacts is:

```bash
./bin/oe speckit:visual <project-slug> all
```

The canonical preview URL is:

```text
http://design-preview:80/design-context/<project-slug>/index.html
```

### Step 3.1: Resolve Artifact Locations

Compute and store the following paths:

```text
design_context_root       = workspace/design-context/<project-slug>/
design_processing_output  = workspace/design-context/<project-slug>/design-processing/
visual_root               = workspace/design-context/<project-slug>/visual-regression/
design_ir                 = workspace/design-context/<project-slug>/design-processing/design-ir.json
prototype_map             = workspace/design-context/<project-slug>/visual-regression/fixtures/prototype-map.json
source_map                = workspace/design-context/<project-slug>/visual-regression/fixtures/source-map.json
screenshots_directory     = workspace/design-context/<project-slug>/visual-regression/screenshots/
route_map                 = workspace/design-context/<project-slug>/visual-regression/fixtures/route-map.json
canonical_preview_url     = http://design-preview:80/design-context/<project-slug>/index.html
handoff_output            = workspace/design-context/<project-slug>/handoff/
```

If config paths are explicitly set, use them. Otherwise use these defaults.

### Step 3.2: Validate design-ir.json

Check whether `design-ir.json` exists at the resolved path.

If found, validate required top-level fields:

- `version` — must be present (number)
- `generated_at` — must be present (ISO timestamp string)
- `project.slug` — must equal `<project-slug>`
- `preview.canonical_url` — must be present; should equal `canonical_preview_url` (record warning if different)
- `source_mode` — must be present
- `render_engine` — must be present
- `viewports` — must be a non-empty array
- `screens` — must be a non-empty array
- `warnings` — must be present (array)

For each screen:

- `id` — must be present
- `name` — must be present
- `kind` — must be present
- `preview_url` — must be present
- `discovery_strategy` — must be present
- `action_path` — must be present
- `variants` — must be a non-empty array
- `warnings` — must be present

For each variant:

- `id` — must be present
- `viewport` — must be present
- `signature` — must be present
- `canvas` — must be present with `width` and `height`
- `screenshot` — must be present
- `screenshot.path` — must resolve to an existing PNG file relative to the screenshots directory
- `screenshot.width` — must be > 0
- `screenshot.height` — must be > 0
- `root_node_id` — must be present
- `nodes` — must be present (array)
- `assets` — must be present (array)
- `warnings` — must be present (array)

Project consistency:

- `design_ir.project.slug` must equal `<project-slug>`
- `design_ir.preview.canonical_url` should equal `canonical_preview_url`
- If canonical URL differs, record a warning rather than failing unless visual artifacts are required.

Record `design-ir.json` status as: `valid`, `invalid:<reason>`, or `missing`.

### Step 3.3: Validate prototype-map.json

Check whether `prototype-map.json` exists at the resolved path.

If found, validate required fields:

- `version` — must be present
- `project.slug` — must equal `<project-slug>`
- `startUrl` — should equal `canonical_preview_url` (record warning if different)
- `discovery.strategy` — must be present
- `pages` — must be a non-empty array
- `warnings` — must be present (array)

For each page:

- `id` — must be present
- `name` — must be present
- `entryLabel` — must be present
- `actionPath` — must be present
- `warnings` — must be present (array)

Record `prototype-map.json` status as: `valid`, `invalid:<reason>`, or `missing`.

### Step 3.4: Validate source-map.json

Check whether `source-map.json` exists at the resolved path.

If found, validate required fields:

- `version` — must be present
- `project.slug` — must equal `<project-slug>`
- `preview.canonicalUrl` — should equal `canonical_preview_url` (record warning if different)
- `screens` — must be a non-empty array
- `viewports` — must be present
- `warnings` — must be present (array)

For each screen:

- `id` — must be present
- `name` — must be present
- `url` — must be present
- `discoveryStrategy` — must be present
- `actionPath` — must be present
- `viewports` — must be a non-empty array

Cross-reference consistency:

- Every source-map screen `id` should exist in design-ir screens
- Every design-ir screen `id` should exist in source-map screens
- Record warnings for any mismatches.

Record `source-map.json` status as: `valid`, `invalid:<reason>`, or `missing`.

### Step 3.5: Validate Screenshots

Check whether `screenshots_directory` exists and contains at least one `.png` file.

For each `variant.screenshot.path` in `design-ir.json`, verify the referenced PNG file exists relative to the screenshots directory.

Record screenshot status: count found, count missing, list of missing references.

### Step 3.6: Handle Missing or Invalid Artifacts

If visual artifact ingestion is enabled and any required artifact is missing or invalid:

**When `--require-visual-artifacts` or `visual.required: true`:**

Fail the command immediately with a clear error:

```text
Error: Workbench visual artifacts are missing or invalid.

Run:
./bin/oe speckit:visual <project-slug> all

Then rerun:
/speckit.open-design.process --project <project-slug>
```

**When visual artifacts are not required (default):**

Record the warning text into all output artifacts:

```
Workbench visual artifacts are missing or invalid.

Run:
./bin/oe speckit:visual <project-slug> all

Then rerun:
/speckit.open-design.process --project <project-slug>
```

Continue processing, but mark `visual_artifacts.available: false` and `visual_artifacts.valid: false` in all generated files.

### Step 3.7: Generate route-map.json

Use `source-map.json` and `design-ir.json` as inputs to generate an editable route map template.

Read existing `route-map.json` if it exists.

**If `route-map.json` does NOT exist:**

Create it with one entry per screen-variant combination discovered from `design-ir.json` and `source-map.json`. Use the following schema:

```json
{
  "version": 1,
  "generated_at": "<iso-timestamp>",
  "project": {
    "slug": "<project-slug>"
  },
  "frontend": {
    "default_url": "http://node-runner:5173"
  },
  "entries": {
    "login-page__desktop": {
      "screenId": "login-page",
      "screenName": "Login Page",
      "viewport": {
        "name": "desktop",
        "width": 1440,
        "height": 1024,
        "deviceScaleFactor": 1
      },
      "implementationRoute": null,
      "referenceScreenshot": "../screenshots/login-page__desktop.png",
      "maxDiffPixels": 200,
      "maxDiffPixelRatio": 0.001,
      "threshold": 0.2,
      "actionPath": [],
      "warnings": [
        "Set implementationRoute after the frontend route exists."
      ]
    }
  },
  "warnings": [
    "implementationRoute values must be filled after frontend routes exist."
  ]
}
```

**If `route-map.json` already exists (and `--force-route-map` is NOT set):**

1. Preserve all existing `implementationRoute` values.
2. Add new entries for screens/variants discovered in `design-ir.json` / `source-map.json` that do not exist in the current route-map.
3. Do NOT remove stale entries.
4. Update viewport dimensions and referenceScreenshot paths for existing entries if they changed.

**If `--force-route-map` is set:**

Regenerate the entire `route-map.json` from `design-ir.json` and `source-map.json`. Discard any existing entries. Write fresh entries with `implementationRoute: null` and the standard warnings.

Use `visual.comparison.diff` thresholds from config for `maxDiffPixels`, `maxDiffPixelRatio`, and `threshold` in each entry.

Write the route-map to `<visual_root>/fixtures/route-map.json`.

### Step 3.8: Update normalized-assets.json

After validating all visual artifacts, add a top-level `visual_artifacts` section to the `normalized-assets.json` produced in Phase 1.

When artifacts are available and valid:

```json
{
  "visual_artifacts": {
    "available": true,
    "valid": true,
    "producer": "opencode-environment:speckit-visual",
    "canonical_url": "http://design-preview:80/design-context/<project-slug>/index.html",
    "design_ir": "design-processing/design-ir.json",
    "prototype_map": "../visual-regression/fixtures/prototype-map.json",
    "source_map": "../visual-regression/fixtures/source-map.json",
    "screenshots_directory": "../visual-regression/screenshots",
    "screen_count": 3,
    "variant_count": 9,
    "screenshot_count": 9,
    "warnings": []
  }
}
```

When artifacts are missing or invalid:

```json
{
  "visual_artifacts": {
    "available": false,
    "valid": false,
    "producer": "opencode-environment:speckit-visual",
    "canonical_url": "http://design-preview:80/design-context/<project-slug>/index.html",
    "warnings": [
      "Workbench visual artifacts are missing or invalid. Run: ./bin/oe speckit:visual <project-slug> all"
    ]
  }
}
```

Path references are relative to the `normalized-assets.json` location (`design-processing/`).

### Step 3.9: Generate frontend-implementation-brief.md

Write `<design_processing_output>/frontend-implementation-brief.md`:

```markdown
# Frontend Implementation Brief

## Design Context

Path:
`workspace/design-context/<project-slug>`

## Canonical Visual Contract

- `design-processing/design-ir.json`
- `visual-regression/fixtures/prototype-map.json`
- `visual-regression/fixtures/source-map.json`
- `visual-regression/screenshots/`

## Visual Comparison

Before backend integration, update:

`visual-regression/fixtures/route-map.json`

Then run from the workbench:

```bash
./bin/oe speckit:visual <project-slug> compare --frontend-url http://node-runner:5173
```

## Static GraphQL Fixture Strategy

Implement the frontend UI first using deterministic static GraphQL JSON fixtures.

Fixture directory recommendation:

`frontend/src/mocks/graphql`

Use one fixture per screen/state.

Do not block visual fidelity work on backend persistence, auth, workflows, or resolver logic.

## Implementation Order

1. Create frontend shell and routing.
2. Create static GraphQL fixture layer.
3. Implement screens from `source-map.json`.
4. Implement components from `component-contracts.md`.
5. Match layout/style from `design-ir.json`.
6. Fill `route-map.json`.
7. Run visual comparison.
8. Iterate until visual diffs pass.
9. Integrate real GraphQL backend.
10. Add complex backend logic.

## Known Warnings

<list-warnings-from-design-ir-and-change-summary>
```

If visual artifacts are missing, include the remediation command and warning at the top of the file.

### Step 3.10: Generate handoff/README.md

Create the handoff directory.

Write `<handoff_output>/README.md`:

```markdown
# Design Context Handoff: <project-slug>

This package contains the canonical design context generated from Open Design sources. It is repository-independent and intended to be consumed by the future frontend implementation.

## Contents

### Source Open Design Export
`sources/open-design-export/` — Original Open Design assets (HTML, CSS, SVGs, metadata). Read-only. These are the authoritative design sources.

### Design Processing Artifacts
`siblings/design-processing/` — Machine-readable and human-readable design extraction:

- `design-ir.json` — Rendered visual implementation contract (consumed from crawler).
- `design-tokens.json` — Extracted colors, typography, spacing, radii, shadows, motion.
- `component-contracts.md` — Detected reusable UI components and their states.
- `page-structures.md` — User-facing screens, pages, flows, and decks.
- `state-variants.yaml` — Design states plus inferred missing states.
- `data-mappings.json` — Schema-to-page data binding points with static fixture strategy.
- `specify-context.md` — Compact context for specification generation.
- `frontend-implementation-brief.md` — Instructions for the frontend implementation agent.
- `change-summary.md` — Processing history.

### Visual Artifacts

Visual artifacts are produced by the workbench static crawler:

```bash
./bin/oe speckit:visual <project-slug> all
```

Produced files:

* `visual-regression/fixtures/prototype-map.json`
* `visual-regression/fixtures/source-map.json`
* `visual-regression/fixtures/route-map.json`
* `visual-regression/screenshots/*.png`
* `design-processing/design-ir.json`

## Final Frontend Comparison

After the frontend implementation exists:

1. Update `visual-regression/fixtures/route-map.json`.
2. Start the frontend.
3. Run:

```bash
./bin/oe speckit:visual <project-slug> compare --frontend-url http://node-runner:5173
```

## How to Consume from a Future Frontend Repository

1. Read `siblings/design-processing/frontend-implementation-brief.md` first.
2. `design-ir.json` defines the pixel-level contract. All views, components, and styles must respect the measured bounds, colors, typography, spacing, shadows, and asset dimensions documented there.
3. Canonical screenshots in `siblings/visual-regression/screenshots/` are the visual acceptance targets.
4. Tokens, components, pages, states, and data mappings inform the implementation.

## Known Warnings

<list-warnings-from-design-ir-and-change-summary>

## Generated

- Generated at: <timestamp>
- Open Design processing extension version: <version>
- Source mode: <source-mode>
- Visual artifact producer: opencode-environment:speckit-visual
```

### Step 3.11: Record Visual Processing Summary

After completing all visual artifact steps, store the following in memory for later phases:

1. Visual ingestion enabled/disabled.
2. Visual artifacts available/valid flags.
3. design-ir.json status (valid/invalid/missing).
4. prototype-map.json status (valid/invalid/missing).
5. source-map.json status (valid/invalid/missing).
6. Screenshot count and missing references.
7. Screen count and variant count from design-ir.json.
8. route-map.json status (created/preserved/updated/regenerated).
9. All warnings collected during validation.
10. Canonical preview URL.

### Step 3.12: Error Handling for Visual Ingestion

- If visual ingestion is disabled, skip this entire phase.
- If `--require-visual-artifacts` or `visual.required: true` is set and any required artifact is missing or invalid, fail the command.
- If artifacts are missing but not required, continue with warnings in all output files.
- If output directories cannot be created, fail the entire command.
- If generated files cannot be written, fail the entire command.
- All captured warnings must appear in `change-summary.md` and the manifest.

## Phase 4: Extract Design Tokens → `design-tokens.json`

Use `design-ir.json` rendered node `computed_style` values as the highest-confidence source when available.

Fall back to structured sources: HTML `<style>`, linked CSS files, component files, SVG attributes, JSON design metadata, and Markdown design-system notes.

Token confidence rules:

| Source | Confidence |
|---|---|
| Rendered computed style from `design-ir.json` | high |
| CSS/static source extraction | medium/high |
| Inferred from names/classes | medium |
| Raster-only inference | low |

Every token derived from `design-ir.json` must include source provenance:

```json
{
  "source": "design-ir",
  "screens": ["login-page", "dashboard"],
  "nodes": ["node-12", "node-45"],
  "confidence": "high"
}
```

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

## Phase 5: Identify Component Contracts → `component-contracts.md`

Use the normalized asset model and `design-ir.json` rendered node trees to detect reusable UI structures.

Priority detection order:

1. Explicit Open Design component metadata, if present.
2. `design-ir.json` rendered node subtrees — repeated DOM subtree shapes, role/tag/class combinations, text/style/bounds patterns, visual clusters, and repeated asset usage across screens.
3. Repeated HTML class names or DOM structures from source files.
4. Component files from MCP or local sources.
5. CSS selectors reused across screens.
6. SVG symbol/id reuse.
7. Markdown/JSON design-system references.
8. Repeated visual asset usage.

Detection heuristics:

- Recurring class names in 2+ files indicate candidate components.
- Identical CSS rules with different class names indicate variants of the same component.
- Repeated DOM subtree shapes across `design-ir.json` screens indicate reusable components.
- Repeated button/input/card/navigation structures in the rendered node tree indicate components.
- Repeated component exports, prop names, or slots indicate reusable components.
- `:hover`, `:focus`, `:active`, `.active`, `.disabled`, `[aria-*]` selectors indicate states.
- Components derived from raster/video-only references must be marked low confidence.

For each component, document:

- Component name, PascalCase.
- Derivation source and confidence level.
- Detection source: `design-ir` when derived from rendered node trees.
- Screens where it appears.
- Representative node ids (from `design-ir.json`) when applicable.
- DOM/role structure.
- Bounds/style summary.
- Typography/colors/spacing/radius/shadow values (from `design-ir.json` computed styles when available).
- States present in designs.
- States absent but required by implementation. Infer states from variants or repeated patterns in `design-ir.json`.
- Props interface, if inferable.
- Content slots.
- Accessibility notes from roles/aria attributes.
- Warnings.

## Phase 6: Map Page, Screen, Flow, and Deck Structures → `page-structures.md`

Document renderable user-facing structures. Use `prototype-map.json` and `source-map.json` as the canonical page list when available.

**Important rule:** For SPA prototype exports, do not treat URL paths as page identity. The canonical page identity is: `screen id + Prototype Map action path + visual signature`.

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

For each prototype page, document:

1. Metadata:
   - Screen id (from `prototype-map.json` or `source-map.json`)
   - Screen name
   - Discovery strategy: `prototype-map` (from `prototype-map.json` discovery.strategy)
   - Prototype Map entry label (from `entryLabel`)
   - Action path (from `actionPath` or `action_path`)
   - Source file (if available from source analysis)
   - Original Open Design path or artifact id, if MCP-sourced
   - Inferred route or identifier
   - App area/domain
   - Authentication requirement, if inferable
   - Confidence
2. Viewports captured (from `design-ir.json` variants):
   - List of viewport names with widths/heights
   - Screenshot references per viewport
   - Design IR variant ids
3. Layout type:
   - app shell
   - centered form/card
   - marketing/landing
   - dashboard/grid
   - detail view
   - list/table view
   - wizard/flow
   - deck/slide
   - media/reference
4. Component tree:
   - component references from `component-contracts.md`
   - data binding points
   - conditional visibility
   - interactive targets
5. Responsive behavior:
   - breakpoints
   - layout shifts
6. Script/interaction notes:
   - toggles
   - filtering
   - form validation
   - navigation
   - animation/motion
7. Accessibility notes:
   - headings
   - landmarks
   - labels
   - focus order, if inferable
8. Warnings.

## Phase 7: Define State Variants → `state-variants.yaml`

State variants come from two sources:

1. Explicit design variants:
   - Companion files such as `dashboard-loading.html`, `login-error.html`, `screen-empty.png`.
   - Open Design metadata naming variants, frames, states, or flows.
   - `prototype-map.json` pages with distinct action paths or entry labels.
   - `source-map.json` screens with different URLs/discovery strategies.
   - `design-ir.json` node states, rendered attributes/classes, aria states, and visible text patterns.

2. Inferred missing states:
   - Only infer states when justified by structure.
   - Mark all inferred states with `shown_in_design: false`.
   - Include the inference rule that produced the state.

Detect and document the following state categories when present or inferable:

| Category | Examples |
|---|---|
| loading | spinners, skeletons, progress indicators |
| empty | zero-state placeholders, "no results" messages |
| error | error messages, failure states |
| success | confirmation messages, completion states |
| disabled | grayed-out controls, non-interactive elements |
| selected | highlighted items, active tabs |
| active | current navigation items, focused elements |
| expanded | open accordions, visible dropdowns |
| collapsed | closed accordions, hidden sections |
| modal | dialogs, overlays, lightboxes |
| drawer | side panels, slide-out menus |
| menu | navigation menus, context menus |
| dropdown | select menus, action menus |
| tooltip | hover tooltips, info popovers |

If a state is required for implementation but absent in the prototype, mark it as inferred.

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

## Phase 8: Generate Data Mappings → `data-mappings.json`

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

Add a `frontend_mocking` section supporting a static-data-first frontend workflow:

```json
{
  "frontend_mocking": {
    "recommended": true,
    "strategy": "static-graphql-json-fixtures-first",
    "purpose": "Decouple visual fidelity from backend complexity.",
    "fixture_directory": "frontend/src/mocks/graphql",
    "notes": [
      "Implement the frontend against deterministic GraphQL JSON fixtures first.",
      "Use one fixture per screen/state discovered from source-map.json.",
      "Keep fixtures aligned with the eventual GraphQL operation shapes.",
      "Only integrate the real backend after visual comparison passes."
    ]
  }
}
```

For each screen in `source-map.json` (or discovered in Phase 1), generate suggested fixture references:

```json
{
  "screenId": "login-page",
  "screenName": "Login Page",
  "suggestedFixture": "frontend/src/mocks/graphql/login-page.query.json",
  "suggestedOperationName": "LoginPageQuery"
}
```

If no schema files are found, still write the file with:

```json
{
  "version": 1,
  "frontend_mocking": { ... },
  "schema_sources": [],
  "pages": {},
  "warnings": ["No schema files found; data mappings were not inferred."]
}
```

## Phase 9: Generate Specification Context → `specify-context.md`

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

- Design IR: `<design-processing-output>/design-ir.json`
- Design tokens: `<design-processing-output>/design-tokens.json`
- Component contracts: `<design-processing-output>/component-contracts.md`
- Page structures: `<design-processing-output>/page-structures.md`
- State variants: `<design-processing-output>/state-variants.yaml`
- Data mappings: `<design-processing-output>/data-mappings.json`
- Frontend implementation brief: `<design-processing-output>/frontend-implementation-brief.md`

## Specification Rules

- Reuse documented components before inventing new UI components.
- Use documented design tokens for colors, typography, spacing, radius, shadow, and motion.
- Preserve documented page structures and interaction targets.
- Include all documented state variants in functional requirements.
- Treat warnings in `change-summary.md` as clarification candidates.

## Pixel Fidelity Rules

- `design-ir.json` is the canonical rendered visual implementation contract. Future frontend implementations MUST preserve measured viewport, bounds, typography, colors, spacing, shadows, radii, z-order, clipping, text metrics, and asset dimensions unless explicitly overridden.
- Canonical screenshots are stored in the sibling `visual-regression/screenshots` directory.
- Once the frontend repository exists, update `visual-regression/fixtures/route-map.json` and run the workbench comparison command.

## Visual Contract

The canonical visual contract is:

`design-processing/design-ir.json`

The prototype pages were discovered by the workbench crawler from:

`http://design-preview:80/design-context/<project-slug>/index.html`

Screens are SPA prototype states, not deep-link URLs. Treat each screen identity as:

`screen id + Prototype Map action path + visual signature`

The frontend implementation MUST match the reference screenshots in:

`visual-regression/screenshots/`

The frontend implementation MUST update:

`visual-regression/fixtures/route-map.json`

after routes exist.

Run comparison from the workbench:

```bash
./bin/oe speckit:visual <project-slug> compare --frontend-url http://node-runner:5173
```

## Recommended Frontend Build Strategy

Build the frontend UI first using static GraphQL JSON fixtures.

Do not wait for complex backend behavior before matching the visual contract.

Recommended sequence:

1. Implement routes and screens with static GraphQL JSON fixtures.
2. Pass visual comparison against the crawled screenshots.
3. Align GraphQL operations with backend schema.
4. Replace mock transport with the real backend.
5. Add complex backend workflows after visual fidelity is stable.

If visual artifacts are missing, include the warning and remediation command at the top of this section.

## Structures Available

| Structure | Type | Source | Required Notes |
|---|---|---|---|

## Changed Since Previous Run

- ...

## Warnings Requiring Human Judgment

- ...
```

## Phase 10: Version History and Change Summary

Generate `change-summary.md` with:

- timestamp
- source mode
- MCP server and project, if used
- source directories
- asset changes since previous run
- generated artifacts
- visual processing summary (if enabled)
- warnings
- confidence summary
- recommended next step

When visual processing is enabled, include a "Workbench Visual Artifact Ingestion" section:

```markdown
## Workbench Visual Artifact Ingestion

- Visual ingestion: enabled
- Visual artifacts available: true/false
- Visual artifacts valid: true/false
- Producer: opencode-environment:speckit-visual
- Canonical preview URL: http://design-preview:80/design-context/<project-slug>/index.html
- design-ir.json: found/missing/invalid
- prototype-map.json: found/missing/invalid
- source-map.json: found/missing/invalid
- screenshots: N found
- screens: N
- variants: N
- route-map.json: created/preserved/updated/regenerated
- warnings: N
```

If visual artifacts are missing, also include the remediation command:

```bash
./bin/oe speckit:visual <project-slug> all
```

If visual processing is disabled, include:

```markdown
## Workbench Visual Artifact Ingestion

- Visual ingestion: disabled
```

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
    "design_ir": "design-ir.json",
    "design_tokens": "design-tokens.json",
    "component_contracts": "component-contracts.md",
    "page_structures": "page-structures.md",
    "state_variants": "state-variants.yaml",
    "data_mappings": "data-mappings.json",
    "frontend_implementation_brief": "frontend-implementation-brief.md",
    "specify_context": "specify-context.md",
    "change_summary": "change-summary.md",
    "prototype_map": "../visual-regression/fixtures/prototype-map.json",
    "source_map": "../visual-regression/fixtures/source-map.json",
    "route_map": "../visual-regression/fixtures/route-map.json",
    "screenshots": "../visual-regression/screenshots",
    "handoff_readme": "../handoff/README.md"
  },
  "visual": {
    "enabled": true,
    "required": false,
    "available": true,
    "valid": true,
    "producer": "opencode-environment:speckit-visual",
    "canonical_url": "http://design-preview:80/design-context/<project-slug>/index.html",
    "screen_count": 0,
    "variant_count": 0,
    "screenshot_count": 0,
    "warnings": []
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

## Phase 11: Final Summary

Report:

```markdown
## Open Design Processing Complete

**Source mode**: `filesystem|mcp|mixed`  
**Source**: `<source directories>`  
**MCP**: `<enabled/disabled, server, project>`  
**Output**: `<design-processing-output>`  
**Specification context**: `<specify-context-path>`  
**Visual processing**: `ingested/disabled`

### Artifacts Generated

| Artifact | File | Contents |
|---|---|---|
| Normalized assets | `normalized-assets.json` | N assets discovered |
| Design tokens | `design-tokens.json` | N colors, M type tokens, K spacing values |
| Component contracts | `component-contracts.md` | N components |
| Page structures | `page-structures.md` | N structures |
| State variants | `state-variants.yaml` | N explicit, M inferred |
| Data mappings | `data-mappings.json` | N mappings / no schema found |
| Frontend impl brief | `frontend-implementation-brief.md` | Handoff instructions |
| Specification context | `specify-context.md` | Compact context for `/speckit.specify` |
| Change summary | `change-summary.md` | Added/modified/removed assets |

### Visual Artifact Ingestion

- Enabled: `yes/no`
- Producer: opencode-environment:speckit-visual
- Canonical URL: `<canonical-preview-url>`
- design-ir.json: found/missing/invalid
- prototype-map.json: found/missing/invalid
- source-map.json: found/missing/invalid
- Screenshots found: N
- Screens: N (from design-ir)
- Variants: N (from design-ir)
- route-map.json: created/preserved/updated/regenerated
- Warnings: N

### Change Summary

- Added: ...
- Modified: ...
- Removed: ...
- Unchanged: ...

### Warnings

- ...

### Next Step

Run `/speckit.specify`. The generated `specify-context.md` should be read and treated as mandatory design context unless explicitly overridden.

If visual artifact ingestion was enabled, ensure the workbench crawler artifacts are current. Run `./bin/oe speckit:visual <project-slug> all` to regenerate if needed.
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
9. Preserve pixel-level fidelity from rendered design IR when available.