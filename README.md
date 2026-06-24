# Spec Kit Open Design Extension

This extension normalizes Open Design assets into global project-level design context for later `/speckit.specify` executions.

## Commands

- `/speckit.open-design.process` — normalize assets, regenerate artifacts, preserve history, generate visual IR and regression package, and write specification context.
- `/speckit.open-design.check` — validate whether normalized artifacts are missing or stale without writing files.
- `/speckit.open-design.draft` — process design context, then create a design-aware draft for `/speckit.specify`.

### Key Arguments

`/speckit.open-design.process` supports these workbench-level and visual arguments:

| Argument | Description |
|---|---|
| `--project <slug>` | Resolve output paths to `workspace/design-context/<slug>/` |
| `--visual` | Enable rendered design IR, canonical screenshots, and visual regression package |
| `--no-visual` | Disable visual processing |
| `--visual-output <path>` | Override visual-regression output directory |
| `--source-map <path>` | Override generated source-map.json location |
| `--route-map <path>` | Override generated route-map.json location |
| `--viewport <name\|WxH>` | Restrict visual extraction to one viewport |
| `--update-screenshots` | Regenerate canonical screenshots |
| `--playwright-tests` | Generate the visual regression package |
| `--no-playwright-tests` | Skip visual regression package generation |
| `--fail-on-visual-warnings` | Fail if visual extraction warnings are emitted |

## Hook

The extension registers a prompted `before_specify` hook:

```yaml
hooks:
  before_specify:
    command: "speckit.open-design.process"
    optional: true
    description: "Normalize Open Design assets before specification generation."
    prompt: "Process Open Design assets before generating the specification?"
```

## Output

### Legacy Mode (no `--project`)

The extension writes global project artifacts under:

```text
designs/design-processing/
```

Main files:

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

### Workbench-Level Mode (`--project <slug>`)

When a project slug is provided, artifacts are organized under a repository-independent design-context package:

```text
workspace/design-context/<slug>/
  source/
    open-design-export/        # Original Open Design assets (read-only)
  design-processing/
    manifest.json
    normalized-assets.json
    design-ir.json             # Canonical rendered visual implementation contract
    design-tokens.json
    component-contracts.md
    page-structures.md
    state-variants.yaml
    data-mappings.json
    frontend-implementation-brief.md
    specify-context.md
    change-summary.md
    history/<timestamp>/...
  visual-regression/
    package.json
    capture-open-design.mjs    # Standalone screenshot capture script
    open-design.visual.spec.ts # Playwright visual comparison spec
    playwright.config.ts
    fixtures/
      source-map.json          # Screen and viewport definitions
      route-map.json           # Screen-to-route mapping (update after routes exist)
    screenshots/               # Canonical reference screenshots
    test-results/
  handoff/
    README.md                  # Human-readable handoff for the frontend repo
```

`design-ir.json` is the canonical rendered visual implementation contract.
`specify-context.md` is the compact, normative file intended for `/speckit.specify` to read.
`frontend-implementation-brief.md` tells future frontend agents how to consume the design context.

## Visual Processing

When `--visual` is enabled (or `visual.enabled: true` in config), the extension generates a repository-independent visual contract that works before a frontend repository exists.

### Workflow

```bash
# Process Open Design assets with visual extraction
/speckit.open-design.process --project demo --visual --playwright-tests
```

This produces a complete visual regression package under `workspace/design-context/demo/visual-regression/`.

### Capture Canonical Screenshots

From the visual-regression directory:

```bash
cd workspace/design-context/demo/visual-regression
npm install
npm run capture
```

Or via the workbench:

```bash
./bin/oe speckit:visual demo capture
```

### Compare Against Frontend

Once the frontend repository exists and `route-map.json` has been updated with real routes:

```bash
cd workspace/design-context/demo/visual-regression
FRONTEND_URL=http://localhost:3000 npx playwright test
```

Or via the workbench:

```bash
./bin/oe speckit:visual demo compare my-frontend-app
```

The visual regression package requires no frontend repository and contains everything needed: a capture script, Playwright test spec, configuration, and fixture files.

## Project-Level Design Context

`--project <slug>` resolves all output paths to `workspace/design-context/<slug>/`:

| Path | Default |
|---|---|
| Open Design sources | `workspace/design-context/<slug>/source/open-design-export/` |
| Design processing | `workspace/design-context/<slug>/design-processing/` |
| Visual regression | `workspace/design-context/<slug>/visual-regression/` |
| Handoff | `workspace/design-context/<slug>/handoff/` |

### Path Resolution Rules

- `--project <slug>` without `--output` → design-context layout
- `--output <path>` → explicit design-processing path (overrides project inference)
- `--visual-output <path>` → explicit visual-regression path (overrides project inference)
- No `--project` and no `--output` → legacy `designs/design-processing/`
- `--visual` without a resolved visual output path → `<design-processing>/../visual-regression`

Existing users who do not use `--project` continue to see the legacy `designs/design-processing/` layout with no visual artifacts.

## Installation

From a spec-kit project:

```bash
specify extension add --dev /path/to/spec-kit-open-design-extension
specify extension list
```

## Configuration

Optional project configuration lives at:

```text
.specify/extensions/open-design/open-design-config.yml
```

Start from `open-design-config.template.yml`.

The template includes these top-level sections:

| Section | Purpose |
|---|---|
| `source` | Design asset directories, include/exclude patterns |
| `output` | Design-processing output paths |
| `project` | Project slug for workbench-level path inference |
| `visual` | Visual extraction, Playwright, viewports, diff thresholds, failure policy |
| `behavior` | Stale policy, history, failure options |
| `mcp` | Open Design MCP server integration |

## Important Behavior

- This extension must not patch core spec-kit files, generated command files, or scripts. It creates and updates only its own generated artifacts under the configured output directory.
- Visual processing is opt-in (`visual.enabled: false` by default). No screenshots, Playwright tests, or visual IR are generated unless explicitly enabled.
- The visual regression package is self-contained and works before any frontend repository exists.
- Source Open Design assets are never mutated.
- All generated paths in manifests are relative to the design-context root when possible.
