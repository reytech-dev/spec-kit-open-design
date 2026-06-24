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
- `--project <slug>`: set the workbench-level project slug. Resolves default output paths to `workspace/design-context/<slug>/`.
- `--visual`: enable visual processing (rendered design IR, canonical screenshots, visual regression package).
- `--no-visual`: disable visual processing even when configured.
- `--visual-output <path>`: override the visual-regression output directory.
- `--source-map <path>`: override the generated source-map.json location.
- `--route-map <path>`: override the generated route-map.json location.
- `--viewport <name|widthxheight>`: restrict visual extraction to one viewport.
- `--update-screenshots`: regenerate canonical screenshots even when design-ir.json exists.
- `--playwright-tests`: generate the visual regression package (package.json, capture script, playwright spec and config).
- `--no-playwright-tests`: skip visual regression package generation.
- `--fail-on-visual-warnings`: fail the command if visual extraction warnings are emitted.

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
design-ir.json
design-tokens.json
component-contracts.md
page-structures.md
state-variants.yaml
data-mappings.json
specify-context.md
frontend-implementation-brief.md
change-summary.md
history/<timestamp>/...
../visual-regression/package.json
../visual-regression/capture-open-design.mjs
../visual-regression/open-design.visual.spec.ts
../visual-regression/playwright.config.ts
../visual-regression/fixtures/source-map.json
../visual-regression/fixtures/route-map.json
../visual-regression/screenshots/
../visual-regression/test-results/
../handoff/README.md
```

`specify-context.md` is the compact, normative artifact that `/speckit.specify` should read and respect.
`design-ir.json` is the canonical rendered visual implementation contract.
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
6. Determine visual processing directories:
   - Resolve whether visual processing is enabled:
     - `--visual` → enabled.
     - `--no-visual` → disabled.
     - Otherwise use configured `visual.enabled` (default: `false`).
   - Determine visual output directory:
     - Use `--visual-output <path>` if provided.
     - Else if `--output` was explicitly provided: `<output>/../visual-regression`.
     - Else if `project.slug` is set: `workspace/design-context/<slug>/visual-regression`.
     - Otherwise use configured `visual.directory`.
     - Default: `null` (visual output only generated when explicitly enabled and a path is resolved).
   - Determine screenshots directory:
     - Defaults to `<visual-output>/screenshots`.
     - Override with configured `visual.screenshots_directory` or `--visual-output`-derived path.
   - Determine source-map location:
     - Use `--source-map <path>` if provided.
     - Otherwise use configured `visual.source_map`.
     - Default: `<visual-output>/fixtures/source-map.json`.
   - Determine route-map location:
     - Use `--route-map <path>` if provided.
     - Otherwise use configured `visual.route_map`.
     - Default: `<visual-output>/fixtures/route-map.json`.
   - Determine handoff output directory:
     - If `project.slug` is set: `workspace/design-context/<slug>/handoff`.
     - Otherwise: `<output-directory>/../handoff`.
   - Resolve playwright test generation:
     - `--playwright-tests` → generate.
     - `--no-playwright-tests` → skip.
     - Otherwise default to `true` when visual processing is enabled.
   - Resolve `--fail-on-visual-warnings` and `visual.failure_policy` settings.
   - Resolve `--viewport` filter if provided.
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

## Phase 3: Generate Rendered Design IR and Visual Regression Package

This phase runs when visual processing is enabled (`--visual` or `visual.enabled: true`).

It generates a repository-independent visual contract, canonical screenshots, and a self-contained Playwright visual regression package under the design-context directory. The phase must complete before semantic artifact generation (tokens, components, pages) so that rendered measurements can inform later phases.

### Step 3.1: Detect Renderable Assets

From the `normalized-assets.json` produced in Phase 1, identify renderable screens:

**Renderable** (will be rendered at viewports):
- `.html`, `.htm` files
- Open Design generated HTML artifacts
- MCP materialized HTML assets
- Explicit frame metadata pointing to renderable output

**Reference-only** (recorded but not rendered unless associated with a renderable screen):
- `.png`, `.jpg`, `.jpeg`, `.webp`, `.svg`, `.pdf`, `.pptx`, `.mp4`, `.mov`

For each renderable asset, generate a stable `screen_id` using this priority:

1. Explicit Open Design frame id from metadata.
2. Explicit page/screen name from Open Design metadata or MCP resource identifier.
3. HTML `<title>` element content, kebab-cased.
4. Route hint inferred from filename (e.g. `login.html` → `login-page`).
5. Normalized relative path from source root, path separators replaced with `-`.
6. First 8 characters of the file content SHA-256 hash.

Store the screen list for subsequent steps.

### Step 3.2: Generate `source-map.json`

Create the visual-regression fixtures directory.

Write `<visual-output>/fixtures/source-map.json`:

```json
{
  "version": 1,
  "generated_at": "<iso-timestamp>",
  "designContextRoot": "..",
  "project": {
    "slug": "<project-slug-or-null>"
  },
  "screens": [
    {
      "id": "login-page",
      "name": "Login Page",
      "sourcePath": "../source/open-design-export/login.html",
      "route": "/source/open-design-export/login.html",
      "sourceHash": "sha256:abc123...",
      "kind": "web-page",
      "viewports": ["desktop", "tablet", "mobile"]
    }
  ],
  "viewports": {
    "desktop": {
      "width": 1440,
      "height": 1024,
      "deviceScaleFactor": 1
    },
    "tablet": {
      "width": 768,
      "height": 1024,
      "deviceScaleFactor": 1
    },
    "mobile": {
      "width": 390,
      "height": 844,
      "deviceScaleFactor": 2
    }
  }
}
```

Rules:
- `sourcePath` must be relative to the source-map.json location, prefixed with `../`.
- `route` is the URL path used when serving statically from design-context root.
- If `--viewport` restricts to one viewport, include only that viewport in each screen's `viewports` array and the top-level `viewports` map.
- Use configured viewport definitions from `visual.viewports`.

### Step 3.3: Generate `route-map.json`

Write `<visual-output>/fixtures/route-map.json`:

```json
{
  "version": 1,
  "generated_at": "<iso-timestamp>",
  "designContextRoot": "..",
  "entries": {
    "login-page__desktop": {
      "screenId": "login-page",
      "viewport": {
        "name": "desktop",
        "width": 1440,
        "height": 1024
      },
      "implementationRoute": "/login",
      "referenceScreenshot": "screenshots/login-page__desktop.png",
      "maxDiffPixels": 200,
      "maxDiffPixelRatio": 0.001,
      "threshold": 0.2
    }
  }
}
```

Rules:
- One entry per screen × viewport combination.
- `implementationRoute` is a best-effort hint. Infer from:
  1. Explicit route metadata in the Open Design asset.
  2. File path pattern (e.g. `login.html` → `/login`, `dashboard.html` → `/dashboard`).
  3. HTML `<title>` or `<meta>` route hints.
  4. Fallback: `/source/open-design-export/<filename>`.
- Mark the file with a comment noting that the frontend agent must update `implementationRoute` once routes are established.
- Use configured diff thresholds from `visual.diff`.

### Step 3.4: Generate Visual Regression `package.json`

Write `<visual-output>/package.json`:

```json
{
  "name": "speckit-open-design-visual-regression",
  "private": true,
  "type": "module",
  "scripts": {
    "capture": "node capture-open-design.mjs",
    "compare": "npx playwright test open-design.visual.spec.ts",
    "update": "npx playwright test open-design.visual.spec.ts --update-snapshots"
  },
  "devDependencies": {
    "@playwright/test": "<version>",
    "playwright": "<version>",
    "http-server": "^14.1.1"
  }
}
```

Use the configured `visual.playwright.package_version` for both `@playwright/test` and `playwright`.

### Step 3.5: Generate `capture-open-design.mjs`

Write `<visual-output>/capture-open-design.mjs`.

This script is the canonical screenshot capture and design IR extraction tool. It must work with only the visual-regression package and its sibling directories — no frontend repository required.

```javascript
import { readFileSync, writeFileSync, mkdirSync, existsSync } from "node:fs";
import { resolve, dirname, join } from "node:path";
import { fileURLToPath } from "node:url";
import { createServer } from "node:http";
import { readFile } from "node:fs/promises";
import { lookup } from "node:mime-types";

const __dirname = dirname(fileURLToPath(import.meta.url));

function loadSourceMap() {
  const path = resolve(__dirname, "fixtures/source-map.json");
  if (!existsSync(path)) {
    console.error("source-map.json not found at", path);
    process.exit(1);
  }
  return JSON.parse(readFileSync(path, "utf-8"));
}

function getMimeType(filePath) {
  const ext = filePath.split(".").pop()?.toLowerCase() || "";
  const mimeMap = {
    html: "text/html", htm: "text/html", css: "text/css",
    js: "application/javascript", mjs: "application/javascript",
    json: "application/json", svg: "image/svg+xml",
    png: "image/png", jpg: "image/jpeg", jpeg: "image/jpeg",
    webp: "image/webp", woff2: "font/woff2", woff: "font/woff",
    ttf: "font/ttf", ico: "image/x-icon",
  };
  return mimeMap[ext] || "application/octet-stream";
}

function startServer(port = 0) {
  const designContextRoot = resolve(__dirname, "..");
  return new Promise((resolveServer) => {
    const server = createServer(async (req, res) => {
      const urlPath = (req.url || "/").split("?")[0];
      const filePath = resolve(designContextRoot, urlPath.replace(/^\//, ""));
      try {
        const content = await readFile(filePath);
        res.writeHead(200, {
          "Content-Type": getMimeType(filePath),
          "Access-Control-Allow-Origin": "*",
        });
        res.end(content);
      } catch {
        res.writeHead(404);
        res.end();
      }
    });
    server.listen(port, "127.0.0.1", () => {
      const addr = server.address();
      resolveServer({ server, port: addr.port });
    });
  });
}

async function capture() {
  const sourceMap = loadSourceMap();
  const { chromium } = await import("playwright");

  const screenshotsDir = resolve(__dirname, "screenshots");
  if (!existsSync(screenshotsDir)) {
    mkdirSync(screenshotsDir, { recursive: true });
  }

  const { server, port } = await startServer(0);
  const baseUrl = `http://127.0.0.1:${port}`;
  console.log(`Static server started on ${baseUrl}`);

  const browser = await chromium.launch({ headless: true });
  const designIR = {
    version: 1,
    generated_at: new Date().toISOString(),
    project: { slug: sourceMap.project?.slug || null },
    source_mode: "filesystem",
    render_engine: {
      name: "playwright",
      browser: "chromium",
      headless: true,
    },
    viewports: Object.entries(sourceMap.viewports).map(([name, v]) => ({
      name,
      ...v,
    })),
    screens: [],
    assets: [],
    warnings: [],
  };

  for (const screen of sourceMap.screens) {
    const screenEntry = {
      id: screen.id,
      name: screen.name,
      source_path: screen.sourcePath,
      source_hash: screen.sourceHash || null,
      kind: screen.kind || "web-page",
      route_hint: screen.route || null,
      confidence: "high",
      variants: [],
      warnings: [],
    };

    for (const vpName of screen.viewports) {
      const variantId = `${screen.id}__${vpName}`;
      const vpDef = sourceMap.viewports[vpName];
      if (!vpDef) continue;

      const page = await browser.newPage();
      await page.setViewportSize({
        width: vpDef.width,
        height: vpDef.height,
      });

      try {
        const url = baseUrl + (screen.route || `/source/open-design-export/${screen.id}.html`);
        await page.goto(url, { waitUntil: "networkidle", timeout: 30000 });
        await page.evaluate(() => document.fonts?.ready);

        const screenshotPath = `${variantId}.png`;
        await page.screenshot({
          path: resolve(screenshotsDir, screenshotPath),
          fullPage: false,
          animations: "disabled",
          caret: "hide",
          scale: "css",
        });

        const nodeTree = await page.evaluate(() => {
          let nodeCounter = 0;
          function extractNode(el, parentId, depth, index) {
            const id = `node-${++nodeCounter}`;
            const tag = (el.tagName || "").toLowerCase();
            const rect = el.getBoundingClientRect?.() || {};
            const style = window.getComputedStyle?.(el) || {};

            const computedStyle = {};
            const styleProps = [
              "display", "position", "boxSizing", "width", "height",
              "margin", "padding", "gap", "color", "backgroundColor",
              "fontFamily", "fontSize", "fontWeight", "lineHeight",
              "letterSpacing", "border", "borderRadius", "boxShadow",
              "opacity", "overflow", "transform", "zIndex",
            ];
            for (const prop of styleProps) {
              const v = style[prop] || style.getPropertyValue?.(prop);
              if (v !== undefined && v !== null && v !== "") {
                computedStyle[prop.replace(/([A-Z])/g, "_$1").toLowerCase()] = v;
              }
            }

            const text = el.childNodes?.length === 1 && el.childNodes[0]?.nodeType === 3
              ? (el.textContent || "").trim()
              : "";

            const attrs = {};
            for (const attr of (el.attributes || [])) {
              attrs[attr.name] = attr.value;
            }

            const children = [];
            let childIndex = 0;
            for (const child of (el.children || [])) {
              children.push(extractNode(child, id, depth + 1, childIndex++));
            }

            return {
              id,
              parent_id: parentId,
              index,
              depth,
              tag,
              name: (attrs["data-component"] || attrs["data-name"] || attrs["aria-label"] || tag)?.toString(),
              type: tag === "img" || tag === "svg" ? (tag === "svg" ? "svg" : "image")
                : tag === "input" || tag === "textarea" || tag === "select" ? "input"
                : tag === "video" ? "image" : text ? "text" : "element",
              role: attrs["role"] || null,
              text: text || null,
              visible: rect.width > 0 && rect.height > 0,
              bounds: {
                x: Math.round(rect.x),
                y: Math.round(rect.y),
                width: Math.round(rect.width),
                height: Math.round(rect.height),
              },
              computed_style: computedStyle,
              attributes: {
                class: attrs["class"] || null,
                id: attrs["id"] || null,
                role: attrs["role"] || null,
                "aria-label": attrs["aria-label"] || null,
                src: attrs["src"] || null,
                href: attrs["href"] || null,
                type: attrs["type"] || null,
              },
              asset_refs: [],
              children,
            };
          }

          const root = document.body || document.documentElement;
          if (!root) return { id: "node-root", parent_id: null, index: 0, depth: 0, tag: "body", children: [], warnings: ["No body element found"] };
          return extractNode(root, null, 0, 0);
        });

        const variant = {
          id: variantId,
          viewport: vpName,
          canvas: {
            width: vpDef.width,
            height: vpDef.height,
            background: "#ffffff",
          },
          screenshot: {
            path: `screenshots/${screenshotPath}`,
            width: vpDef.width,
            height: vpDef.height,
            device_scale_factor: vpDef.deviceScaleFactor || 1,
          },
          root_node_id: nodeTree.id,
          nodes: [nodeTree],
          assets: [],
          warnings: [],
        };

        screenEntry.variants.push(variant);
      } catch (err) {
        const warning = `Render failed for ${variantId}: ${err.message}`;
        console.error(warning);
        screenEntry.warnings.push(warning);
        designIR.warnings.push(warning);
      } finally {
        await page.close();
      }
    }

    designIR.screens.push(screenEntry);
  }

  await browser.close();
  server.close();

  const irPath = resolve(__dirname, "..", "design-processing", "design-ir.json");
  writeFileSync(irPath, JSON.stringify(designIR, null, 2));
  console.log(`design-ir.json written to ${irPath}`);
  console.log(`Screens rendered: ${designIR.screens.length}`);
  console.log(
    `Variants captured: ${designIR.screens.reduce((c, s) => c + s.variants.length, 0)}`
  );
  console.log(`Warnings: ${designIR.warnings.length}`);
  if (designIR.warnings.length > 0) {
    console.log("Warnings:");
    designIR.warnings.forEach((w) => console.log(`  - ${w}`));
  }
}

capture().catch((err) => {
  console.error("Capture failed:", err);
  process.exit(1);
});
```

### Step 3.6: Generate `open-design.visual.spec.ts`

Write `<visual-output>/open-design.visual.spec.ts`:

```typescript
import { test, expect, Page } from "@playwright/test";
import { readFileSync, existsSync } from "node:fs";
import { resolve, dirname } from "node:path";
import { fileURLToPath } from "node:url";

const __dirname = dirname(fileURLToPath(import.meta.url));

interface RouteEntry {
  screenId: string;
  viewport: {
    name: string;
    width: number;
    height: number;
  };
  implementationRoute: string;
  referenceScreenshot: string;
  maxDiffPixels: number;
  maxDiffPixelRatio: number;
  threshold: number;
}

interface RouteMap {
  entries: Record<string, RouteEntry>;
}

let routeMap: RouteMap;
let frontendUrl: string;

test.describe("Open Design Visual Regression", () => {
  test.beforeAll(() => {
    const routeMapPath = resolve(__dirname, "fixtures", "route-map.json");
    if (!existsSync(routeMapPath)) {
      throw new Error(
        `route-map.json not found at ${routeMapPath}. ` +
        `Run capture first: node capture-open-design.mjs`
      );
    }
    routeMap = JSON.parse(readFileSync(routeMapPath, "utf-8"));

    frontendUrl = process.env.FRONTEND_URL || "";
    if (!frontendUrl) {
      console.warn(
        "FRONTEND_URL is not set. Visual comparison will likely fail.\n" +
        "Set FRONTEND_URL to the running frontend server URL, e.g.:\n" +
        "  FRONTEND_URL=http://localhost:3000 npx playwright test"
      );
    }
  });

  for (const [entryId, entry] of Object.entries(routeMap.entries || {})) {
    test(`Visual comparison: ${entryId}`, async ({ page }: { page: Page }) => {
      test.skip(
        !frontendUrl,
        "FRONTEND_URL is not set. Skipping visual comparison."
      );

      await page.setViewportSize({
        width: entry.viewport.width,
        height: entry.viewport.height,
      });

      const route = entry.implementationRoute;
      const targetUrl = frontendUrl.replace(/\/$/, "") + route;
      await page.goto(targetUrl, {
        waitUntil: "networkidle",
        timeout: 30000,
      });

      await page.evaluate(() => document.fonts?.ready);

      const screenshotPath = resolve(
        __dirname,
        entry.referenceScreenshot
      );

      await expect(page).toHaveScreenshot(screenshotPath, {
        fullPage: false,
        maxDiffPixels: entry.maxDiffPixels,
        maxDiffPixelRatio: entry.maxDiffPixelRatio,
        threshold: entry.threshold,
        animations: "disabled",
        caret: "hide",
        scale: "css",
      });
    });
  }
});
```

### Step 3.7: Generate `playwright.config.ts`

Write `<visual-output>/playwright.config.ts`:

```typescript
import { defineConfig } from "@playwright/test";
import { resolve, dirname } from "node:path";
import { fileURLToPath } from "node:url";

const __dirname = dirname(fileURLToPath(import.meta.url));

export default defineConfig({
  testDir: __dirname,
  outputDir: resolve(__dirname, "test-results"),
  snapshotDir: resolve(__dirname, "screenshots"),
  fullyParallel: false,
  retries: 0,
  workers: 1,
  reporter: [["list"], ["html", { outputFolder: resolve(__dirname, "test-results", "html-report") }]],
  use: {
    browserName: "chromium",
    headless: true,
    viewport: null,
    ignoreHTTPSErrors: true,
    screenshot: "off",
  },
  projects: [
    {
      name: "chromium",
      use: { browserName: "chromium" },
    },
  ],
});
```

### Step 3.8: Generate `design-ir.json`

`design-ir.json` is the canonical rendered visual implementation contract.

The complete schema is:

```json
{
  "version": 1,
  "generated_at": "<iso-timestamp>",
  "project": {
    "slug": "<project-slug>"
  },
  "source_mode": "filesystem|mcp|mixed",
  "render_engine": {
    "name": "playwright",
    "browser": "chromium",
    "headless": true
  },
  "viewports": [
    {
      "name": "desktop",
      "width": 1440,
      "height": 1024,
      "device_scale_factor": 1
    }
  ],
  "screens": [
    {
      "id": "login-page",
      "name": "Login Page",
      "source_path": "source/open-design-export/login.html",
      "source_hash": "sha256:abc123...",
      "kind": "web-page",
      "route_hint": "/login",
      "confidence": "high",
      "variants": [
        {
          "id": "login-page__desktop",
          "viewport": "desktop",
          "canvas": {
            "width": 1440,
            "height": 1024,
            "background": "#ffffff"
          },
          "screenshot": {
            "path": "../visual-regression/screenshots/login-page__desktop.png",
            "width": 1440,
            "height": 1024,
            "device_scale_factor": 1
          },
          "root_node_id": "node-1",
          "nodes": [],
          "assets": [],
          "warnings": []
        }
      ],
      "warnings": []
    }
  ],
  "assets": [],
  "warnings": []
}
```

Node schema (each entry in the `nodes` array):

```json
{
  "id": "node-123",
  "parent_id": "node-1",
  "index": 0,
  "depth": 2,
  "tag": "button",
  "name": "Primary Button",
  "type": "element|text|image|svg|input|component|container",
  "role": "button",
  "text": "Continue",
  "visible": true,
  "bounds": {
    "x": 540,
    "y": 712,
    "width": 360,
    "height": 48
  },
  "computed_style": {
    "display": "flex",
    "position": "relative",
    "box_sizing": "border-box",
    "width": "360px",
    "height": "48px",
    "margin": "0px",
    "padding": "0px 16px",
    "gap": "8px",
    "color": "rgb(255, 255, 255)",
    "background_color": "rgb(0, 87, 255)",
    "font_family": "Inter, sans-serif",
    "font_size": "16px",
    "font_weight": "600",
    "line_height": "24px",
    "letter_spacing": "normal",
    "border": "0px none rgb(255, 255, 255)",
    "border_radius": "8px",
    "box_shadow": "none",
    "opacity": "1",
    "overflow": "visible",
    "transform": "none",
    "z_index": "auto"
  },
  "attributes": {
    "class": "primary-button",
    "id": null,
    "role": "button",
    "aria-label": null,
    "src": null,
    "href": null,
    "type": "submit"
  },
  "asset_refs": [],
  "children": []
}
```

**Initial generation (without Playwright capture):**

Write a skeleton `design-ir.json` with:

- `version`, `generated_at`, `project`, `source_mode`, `render_engine` populated.
- `viewports` populated from config.
- `screens` populated with id, name, source_path, source_hash, kind, route_hint, confidence from the renderable assets list (without variants).
- `warnings` set to `["design-ir.json generated from source analysis only; run capture-open-design.mjs for full rendered extraction"]`.

**Full generation (via capture-open-design.mjs):**

If Playwright is available and `capture-open-design.mjs` executes successfully, the script writes the complete `design-ir.json` with fully populated variants, node trees, computed styles, and bounds as described above.

**When to attempt capture execution:**

Attempt to execute `node <visual-output>/capture-open-design.mjs` when:
- `--update-screenshots` is present, OR
- `design-ir.json` does not exist, OR
- `design-ir.json` exists but has the "source analysis only" warning, OR
- `--force` is present.

Skip capture execution when `--no-playwright-tests` is present or Playwright is not installed.

If capture fails:
- Keep the skeleton `design-ir.json`.
- Record the failure and errors in `design-ir.json` `warnings` array.
- Record the failure in `change-summary.md`.
- Do NOT fail the entire processing command unless `--fail-on-visual-warnings` or `visual.failure_policy.fail_on_render_error: true`.

### Step 3.9: Generate `frontend-implementation-brief.md`

Write `<design-processing-output>/frontend-implementation-brief.md`:

```markdown
# Frontend Implementation Brief

This document tells future frontend agents how to consume the design context generated
by spec-kit-open-design. It is written before a frontend repository exists and is intended
to be read by the agent that first creates the frontend implementation.

## Design Context Location

The canonical design context lives in this directory:

\`\`\`text
<design-context-root>/
  source/open-design-export/    # Original Open Design assets (read-only)
  design-processing/            # Generated specification artifacts
  visual-regression/            # Visual regression package
  handoff/                      # Human-readable handoff
\`\`\`

## Canonical Artifacts

The following artifacts are normative and must be respected by any frontend implementation:

| Artifact | File | Purpose |
|---|---|---|
| Design IR | `design-processing/design-ir.json` | Canonical rendered visual implementation contract. Defines viewport-level pixel fidelity, node trees, computed styles, bounds, and measurements. |
| Canonical screenshots | `visual-regression/screenshots/` | Per-viewport reference images. All frontend implementations MUST match these within configured diff thresholds. |
| Design tokens | `design-processing/design-tokens.json` | Extracted CSS custom properties, colors, typography, spacing, radii, shadows, and motion values. |
| Component contracts | `design-processing/component-contracts.md` | Reusable UI components detected from design sources. |
| Page structures | `design-processing/page-structures.md` | User-facing page, screen, flow, and deck structures. |
| State variants | `design-processing/state-variants.yaml` | States present in designs plus inferred missing states. |
| Data mappings | `design-processing/data-mappings.json` | Schema-to-page data binding points. |
| Spec context | `design-processing/specify-context.md` | Compact normative context for `/speckit.specify`. |
| Change summary | `design-processing/change-summary.md` | What changed since the last design processing run. |

## Visual Implementation Contract

`design-ir.json` is the single source of truth for pixel-level fidelity. It contains:

- **Viewport definitions**: exact width, height, and device scale factor for each target viewport.
- **Screen variants**: one per screen × viewport combination.
- **Node trees**: full DOM tree with computed CSS styles, bounding boxes, visibility, roles, and text content.
- **Screenshot references**: paths to canonical screenshots.

Every future frontend test should compare its rendered output against this contract.

## Route Map

The file `visual-regression/fixtures/route-map.json` maps each (screen, viewport) combination to an expected implementation route. **This file must be updated once the frontend repository is created and routes are established.**

Each entry looks like:

\`\`\`json
"login-page__desktop": {
  "implementationRoute": "/login",
  ...
}
\`\`\`

Update `implementationRoute` to match the actual route path used in the frontend application.

## Running Visual Comparison

Once the frontend repository exists and routes are configured, run visual comparison through the workbench:

1. **Capture canonical screenshots** (from Open Design sources):
   \`\`\`bash
   ./bin/oe speckit:visual <project-slug> capture
   \`\`\`

2. **Compare frontend against canonical screenshots**:
   \`\`\`bash
   ./bin/oe speckit:visual <project-slug> compare <frontend-repo-path>
   \`\`\`

Or run directly from the visual-regression directory:

\`\`\`bash
cd workspace/design-context/<project-slug>/visual-regression
npm install
FRONTEND_URL=http://localhost:3000 npx playwright test
\`\`\`

## Updating Canonical Screenshots

If the design sources change, regenerate:

\`\`\`bash
./bin/oe speckit:visual <project-slug> capture --update-screenshots
\`\`\`

## Known Warnings

<list-warnings-from-design-ir-and-change-summary>
```

### Step 3.10: Generate `handoff/README.md`

Create the handoff directory.

Write `<handoff-directory>/README.md`:

```markdown
# Design Context Handoff: <project-slug>

This package contains the canonical design context generated from Open Design sources. It is repository-independent and intended to be consumed by the future frontend implementation.

## Contents

### Source Open Design Export
`sources/open-design-export/` — Original Open Design assets (HTML, CSS, SVGs, metadata). Read-only. These are the authoritative design sources.

### Design Processing Artifacts
`siblings/design-processing/` — Machine-readable and human-readable design extraction:

- `design-ir.json` — Rendered visual implementation contract (pixel fidelity, node trees, styles, bounds).
- `design-tokens.json` — Extracted colors, typography, spacing, radii, shadows, motion.
- `component-contracts.md` — Detected reusable UI components and their states.
- `page-structures.md` — User-facing screens, pages, flows, and decks.
- `state-variants.yaml` — Design states plus inferred missing states.
- `data-mappings.json` — Schema-to-page data binding points.
- `specify-context.md` — Compact context for specification generation.
- `frontend-implementation-brief.md` — Instructions for the frontend implementation agent.
- `change-summary.md` — Processing history.

### Visual Regression Package
`siblings/visual-regression/` — Self-contained Playwright visual regression:

- `package.json` — npm dependencies and scripts.
- `capture-open-design.mjs` — Canonical screenshot capture from Open Design sources.
- `open-design.visual.spec.ts` — Visual comparison test spec.
- `playwright.config.ts` — Playwright configuration.
- `fixtures/source-map.json` — Screen and viewport definitions.
- `fixtures/route-map.json` — Screen-to-route mapping (update after frontend routes exist).
- `screenshots/` — Canonical reference screenshots.

## How to Consume from a Future Frontend Repository

1. Read `siblings/design-processing/frontend-implementation-brief.md` first.
2. `design-ir.json` defines the pixel-level contract. All views, components, and styles must respect the measured bounds, colors, typography, spacing, shadows, and asset dimensions documented there.
3. Canonical screenshots in `siblings/visual-regression/screenshots/` are the visual acceptance targets.
4. Tokens, components, pages, states, and data mappings inform the implementation.

## How to Update `route-map.json`

After creating the frontend repository and establishing routes, edit:

```
siblings/visual-regression/fixtures/route-map.json
```

Update each entry's `implementationRoute` to match the actual route used in the application (e.g. `/login`, `/dashboard`, `/settings/profile`).

## How to Run Comparison

From the workbench:

```bash
./bin/oe speckit:visual <project-slug> compare <frontend-repo-path>
```

Or from the design-context root:

```bash
cd workspace/design-context/<project-slug>/visual-regression
npm install
FRONTEND_URL=http://localhost:3000 npx playwright test
```

## Known Warnings

<list-warnings-from-design-ir-and-change-summary>

## Generated

- Generated at: <timestamp>
- Open Design processing extension version: <version>
- Source mode: <source-mode>
```

### Step 3.11: Add Visual Artifacts to Manifest and Summaries

After generating all visual artifacts, update the in-memory tracking for later phases:

1. Record visual artifact paths for manifest.json (Phase 10).
2. Record visual processing summary for change-summary.md (Phase 10).
3. Record pixel fidelity rules for specify-context.md (Phase 9).
4. Record visual processing information for the final summary (Phase 11).

### Step 3.12: Error Handling for Visual Processing

- If visual processing is disabled, skip this entire phase.
- If a single screen fails to render, record a warning and continue with remaining screens.
- If all screens fail, write a skeleton `design-ir.json` with warnings and continue.
- If output directories cannot be created, fail the entire command.
- If generated files cannot be written, fail the entire command.
- If `--fail-on-visual-warnings` is present and any warning exists, fail the command after generating all possible artifacts.
- If `visual.failure_policy.fail_on_render_error` is true and any render error occurs, fail the command.
- All captured warnings must also appear in `design-ir.json` `warnings` array.

## Phase 4: Extract Design Tokens → `design-tokens.json`

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

## Phase 5: Identify Component Contracts → `component-contracts.md`

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

## Phase 6: Map Page, Screen, Flow, and Deck Structures → `page-structures.md`

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

## Phase 7: Define State Variants → `state-variants.yaml`

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

If no schema files are found, still write the file with:

```json
{
  "version": 1,
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
- The generated visual regression package defines the acceptance workflow.
- Once the frontend repository exists, update `visual-regression/fixtures/route-map.json` and run the workbench comparison command.

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

When visual processing is enabled, include a "Visual Processing" section:

```markdown
## Visual Processing

- Visual processing: enabled
- Project slug: `<project-slug>`
- Design context root: `<path>`
- Visual regression directory: `<path>`
- Screens rendered: N
- Screenshots generated: N
- Viewports: desktop, tablet, mobile
- Playwright package: `<version>`
- Warnings: N
```

If visual processing is disabled, include:

```markdown
## Visual Processing

- Visual processing: disabled
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
    "visual_regression": "../visual-regression/open-design.visual.spec.ts",
    "visual_source_map": "../visual-regression/fixtures/source-map.json",
    "visual_route_map": "../visual-regression/fixtures/route-map.json",
    "handoff_readme": "../handoff/README.md"
  },
  "visual": {
    "enabled": false,
    "directory": "../visual-regression",
    "rendered_screen_count": 0,
    "screenshot_count": 0,
    "viewports": [],
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
**Visual processing**: `enabled/disabled`

### Artifacts Generated

| Artifact | File | Contents |
|---|---|---|
| Normalized assets | `normalized-assets.json` | N assets discovered |
| Design IR | `design-ir.json` | N screens, M viewport variants |
| Design tokens | `design-tokens.json` | N colors, M type tokens, K spacing values |
| Component contracts | `component-contracts.md` | N components |
| Page structures | `page-structures.md` | N structures |
| State variants | `state-variants.yaml` | N explicit, M inferred |
| Data mappings | `data-mappings.json` | N mappings / no schema found |
| Frontend impl brief | `frontend-implementation-brief.md` | Handoff instructions |
| Specification context | `specify-context.md` | Compact context for `/speckit.specify` |
| Change summary | `change-summary.md` | Added/modified/removed assets |

### Visual Processing

- Enabled: `yes/no`
- Visual regression: `<visual-output>`
- Screens rendered: N
- Screenshots captured: N
- Viewports: desktop, tablet, mobile
- Playwright tests: `generated/skipped`
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

If visual processing was enabled, the visual regression package at `<visual-output>` is ready for use. Run `./bin/oe speckit:visual <project-slug> capture` to generate canonical screenshots, or `npm install && npm run capture` from the visual-regression directory.
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