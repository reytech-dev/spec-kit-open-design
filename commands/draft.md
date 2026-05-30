---
description: "Process Open Design context, then create design-aware draft input for speckit.specify."
---

# Open Design Draft Command

## User Input

```text
$ARGUMENTS
```

Consider user input before proceeding. If the input is empty, ask the user for a short description of the feature they want to specify.

Supported arguments:

- `--source <path>`: pass through to `/speckit.open-design.process`.
- `--output <path>`: pass through to `/speckit.open-design.process`.
- `--force`: pass through to `/speckit.open-design.process`.
- `--diff`: pass through to `/speckit.open-design.process`.
- `--no-history`: pass through to `/speckit.open-design.process`.
- `--no-mcp`: pass through to `/speckit.open-design.process`.
- `--mcp-project <name-or-id>`: pass through to `/speckit.open-design.process`.
- `--mcp-materialize <path>`: pass through to `/speckit.open-design.process`.
- `--mcp-required`: pass through to `/speckit.open-design.process`.

All remaining text is the feature idea to draft.

## Purpose

Create a design-aware, implementation-neutral feature draft by first normalizing Open Design assets into project-level context, then drafting polished input for `/speckit.specify`.

This command is an orchestration helper. It exists to avoid relying on the ordering of multiple optional `before_specify` hooks.

## Safety Rules

- Do not patch `.specify/scripts/*`.
- Do not patch installed `speckit.specify`, `speckit.plan`, `speckit.tasks`, or other core command files.
- Do not mutate source design assets.
- Do not mutate the Open Design project through MCP.
- Do not create a Spec Kit feature directory.
- Do not write `spec.md`.
- Do not run Spec Kit feature-generation scripts.
- Only `/speckit.open-design.process` may write or update generated design artifacts under the configured output directory.
- The final draft must remain implementation-neutral unless a technical detail is part of the product contract.

## Phase 1: Process Open Design Context

Run the behavior of `/speckit.open-design.process` first.

Pass through supported processing flags:

- `--source`
- `--output`
- `--force`
- `--diff`
- `--no-history`
- `--no-mcp`
- `--mcp-project`
- `--mcp-materialize`
- `--mcp-required`

After processing, verify whether the configured specification context file exists.

Default:

```text
designs/design-processing/specify-context.md
```

If the file is missing:

- Continue only when Open Design processing reported that no assets were available and configuration allows that.
- Otherwise report that Open Design context could not be prepared and ask the user to run `/speckit.open-design.process --force --diff`.

## Phase 2: Read Design Context

Read `designs/design-processing/specify-context.md` if present.

Treat it as normative project design context for the draft.

Use it to influence:

- user-visible state coverage
- interaction expectations
- responsive behavior
- accessibility requirements
- visible content structure
- reuse of established flows or patterns
- warnings that should become clarification candidates

Do not copy low-level implementation details into the final draft unless they define observable product behavior.

Avoid naming implementation artifacts such as:

- component names
- token names
- CSS classes
- file paths
- framework names
- GraphQL operation names
- database tables

## Phase 3: Draft Feature Flow Text

Apply the behavior of `/speckit.draft.create` to the remaining feature idea.

When design context exists, include design-driven product constraints only when they affect observable behavior, user experience, acceptance criteria, states, accessibility, localization, or responsiveness.

If the feature idea conflicts with the design context:

1. Prefer explicit user input.
2. Record the design deviation as a non-blocking assumption if it does not materially change scope.
3. Ask a clarification question only when the conflict materially changes product behavior, privacy/security behavior, acceptance scenarios, or downstream planning.

## Output Format

When ready, respond with:

1. A short note indicating whether any important assumptions remain.
2. A fenced `text` block containing only the final flow text to pass to `/speckit.specify`.