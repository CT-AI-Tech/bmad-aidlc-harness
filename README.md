# AI-DLC Compliance Harness — BMAD native module

This folder is a real, installable BMAD module (`module.yaml` at its root, per
BMAD's own module schema) — not a set of files to clone-and-copy by hand. It's meant
to be the entire content of a dedicated release branch; see the harness repo's
`docs/` for how this fits into the wider project.

## Install

```bash
bmad-method install --custom-source <git-url-to-this-branch> --modules bmm,ai-dlc-compliance --tools claude-code
```

This does three things natively, with zero manual file copying:
1. Clones the branch via git (works against a **private** repo — confirmed hands-on;
   BMAD's installer shells out to git and inherits your existing credentials, unlike
   Speckit's URL-based install which cannot).
2. Prompts for `harness_path`, `vertical`, and `block_on_binding` — the same three
   decisions every other adapter in this harness asks for, just asked natively during
   install instead of by editing a YAML file afterward. Saved to
   `_bmad/config.toml` under `[modules.ai-dlc-compliance]`.
3. Installs `ai-dlc-compliance-check` into `.claude/skills/` (or whichever tool's
   target directory you passed to `--tools`).

## The one step this can't do for you

BMAD has no mechanism for one module to write another module's workflow-customization
files — `activation_steps_prepend` on `bmad-product-brief`, `bmad-architecture`,
`bmad-create-epics-and-stories`, and `bmad-code-review` lives in **those** workflows'
own `_bmad/custom/` surface, which only a human (or a script outside BMAD's module
system) can populate. The installer prints this as a `post_install_message`; here's
the actual command:

```bash
cp overrides/*.toml _bmad/custom/
```

(Run from wherever `--custom-source` cloned this module to, or re-fetch just these 4
files from the branch if that clone wasn't kept.)

## What each override does

| File | BMAD workflow | Harness phase it loads |
|---|---|---|
| `bmad-product-brief.toml` | `bmad-product-brief` | `constitution` |
| `bmad-architecture.toml` | `bmad-architecture` | `development` |
| `bmad-create-epics-and-stories.toml` | `bmad-create-epics-and-stories` | `development` (re-check) |
| `bmad-code-review.toml` | `bmad-code-review` | `review` — runs as one more parallel reviewer |

## Verify it landed

```bash
python _bmad/scripts/resolve_config.py --project-root . | grep -A4 ai-dlc-compliance
python _bmad/scripts/resolve_customization.py --skill .claude/skills/bmad-architecture --project-root . --key workflow.activation_steps_prepend
```

The first confirms your `harness_path`/`vertical` answers saved correctly. The second
only shows the override once you've done the manual `cp` above — same verification
method used throughout this harness's build.

## Honest status

Structure and config-prompting verified hands-on against a local path. Verifying the
full flow against a **real** git branch URL (not a local path) and against a public
vs. private repo is the next real test — see the harness repo's adoption guide for
the broader context this module fits into.
