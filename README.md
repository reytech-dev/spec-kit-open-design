# Spec Kit Open Design Extension

This extension normalizes Open Design assets into global project-level design context for later `/speckit.specify` executions.

## Commands

- `/speckit.open-design.process` — normalize assets, regenerate artifacts, preserve history, and write specification context.
- `/speckit.open-design.check` — validate whether normalized artifacts are missing or stale without writing files.

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

`specify-context.md` is the compact, normative file intended for `/speckit.specify` to read.

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

## Important behavior

This extension must not patch core spec-kit files, generated command files, or scripts. It creates and updates only its own generated artifacts under the configured output directory.
