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

## You don't need to clone ai-dlc-harness yourself

`harness_path` no longer requires a manual `git clone` as a sibling folder. The
`ai-dlc-compliance-check` skill auto-fetches it (git clone, falling back to a zip
download) into `harness_path` the first time it runs, if nothing is there yet. It
never touches a path that already has something in it, so pointing `harness_path` at
an existing checkout (a fork, a pinned version, a private mirror) is still fully
supported — auto-fetch only fires when the folder is genuinely empty.

## Honest status

Verified hands-on end-to-end against the real public repos (not local paths):
module discovery, config-prompting, and skill install all confirmed against
`https://github.com/CT-AI-Tech/bmad-aidlc-harness`, with `harness_path` resolving to
a real clone of `https://github.com/CT-AI-Tech/ai-dlc-harness`. The auto-fetch
behavior above is implemented as skill instructions (this harness's agents always run
with shell access, same as every other step here) rather than a BMAD-native
installer hook — see `docs/MANUAL-TEST-GUIDE.md` in the main harness repo for the
full walkthrough.
