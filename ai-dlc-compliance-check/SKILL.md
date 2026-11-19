---
name: ai-dlc-compliance-check
description: Load only the AI-DLC harness guardrails relevant to the current vertical and BMAD workflow phase, and surface binding/advisory findings. Invoked as an activation_steps_prepend entry from BMAD workflow overrides - use when invoked by name.
---

# AI-DLC Compliance Check (BMAD native module)

Invoked as `skill:ai-dlc-compliance-check phase=<constitution|development|review>`
from a BMAD workflow's `activation_steps_prepend` (set via the override files this
module ships in `overrides/` - see this module's README for the one manual step that
copies them into `_bmad/custom/`).

This is the module-native variant of `ai-dlc-compliance-check`. The only real
difference from the manually-installed version: **config comes from `_bmad/config.toml`
now, not a separate `compliance-config.yml`** - because installing via a real BMAD
module means `harness_path`, `vertical`, and `block_on_binding` were already asked for
and saved during `bmad-method install`, the same way BMAD's own `core` and `bmm`
modules ask for `user_name` or `planning_artifacts`.

## Step 1 — Load config, degrade gracefully if absent

```
uv run {project-root}/_bmad/scripts/resolve_config.py --project-root {project-root}
```
(falls back to plain `python` if `uv` isn't on PATH, same as every other BMAD skill
in this harness).

Read `{modules.ai-dlc-compliance.harness_path}`, `{modules.ai-dlc-compliance.vertical}`,
`{modules.ai-dlc-compliance.block_on_binding}` from the result.

- Missing (module not installed, or `_bmad/config.toml` unreadable) → skip silently
  with a one-line note. Never fail the calling workflow over the harness's own setup.
- `harness_path` unresolvable → warn once, skip, never fail the calling workflow.

Evidence directory is not a prompted config value (keeping the install to 3 questions,
not more) - it defaults to `_bmad-output/compliance`, matching BMAD's own
`output_folder` convention.

## Step 2 — Determine the phase from the invocation argument

Identical to the manually-installed variant - the phase is passed explicitly on the
`skill:` directive itself, because BMAD's `activation_steps_prepend` entries are plain
instruction strings, not hook events with a name attached:

| Override file (in `_bmad/custom/`, copied from this module's `overrides/`) | `phase=` argument | Harness phase |
|---|---|---|
| `bmad-product-brief.toml` | `constitution` | `constitution` |
| `bmad-architecture.toml` | `development` | `development` |
| `bmad-create-epics-and-stories.toml` | `development` | `development` (re-check) |
| `bmad-code-review.toml` | `review` | `review` |

## Step 3 — Query the index, not the guardrail content

Load `<harness_path>/registry/guardrail-index.yaml`, filter on
`vertical == <configured vertical> OR "cross-cutting"` and
`lifecycle_phase == <resolved phase> OR "cross-cutting"`, intersect
`applicability_tags` against tags derived from the artifact in front of you (the
product brief draft, the architecture spine, the epics/stories list, or the diff under
review). Surface the whole narrowed set if tags don't clearly discriminate further.

## Step 4 — Load only the matched guardrail files

Open only the matched `path` values — typically 1-4 files, never the whole vertical.

## Step 5 — Apply binding_tier

- **`binding`**: hard constraint. If `block_on_binding == "true"`, stop and ask the
  user to resolve each blocking finding before the workflow's own work begins — this
  runs as an `activation_steps_prepend` step, so a block here stops the workflow
  before its real work starts, not after.
- **`advisory`**: flagged note, never blocks.

## Step 6 — Record evidence

Append to `_bmad-output/compliance/<feature-or-project-slug>/guardrails-applied.md`.

## Step 7 — Report

Print the report *before* the BMAD agent's own greeting, since this often runs as an
`activation_steps_prepend` entry — a blocking finding should be the first thing the
user sees, not buried after the workflow has already started talking.

## Graceful degradation

Identical to every other adapter in this harness: missing config, unresolvable
`harness_path`, or an unparsable `guardrail-index.yaml` all skip with a warning and
never fail the calling BMAD workflow for this skill's own infrastructure problems.
