---
name: tidy
description: Run tofu fmt and pre-commit against only the folders touched in this session, and clean up any linting issues found. Use when the user asks to tidy up, clean up, or finalize changes before committing/PRing.
---

## Overview

`tidy` is a wrap-up step: run it after a batch of edits, right before committing or
opening a PR. It formats and lints only the directories that actually changed in
this session — not the whole repo — so it stays fast and doesn't touch unrelated
code.

Invocation: `/tidy`

## Step 1 — Determine Scope

Scope is **the set of directories containing files edited during this session**,
not the whole repo and not `git diff` against a base branch (session edits may
already be committed, or may span multiple unrelated changes you don't want to
touch).

1. Recall which files you (the agent) created or modified this session via
   `fs_write`, `str_replace`, `fs_append`, `smart_relocate`, etc.
2. If session memory is unclear or was compacted, fall back to:
   - `git status --short` (uncommitted changes), and
   - `git diff HEAD~N --stat` only if the user confirms which commits belong to
     this session.
3. Reduce the file list to a set of unique containing directories. Don't run
   against the repo root unless a root-level file (e.g. `.pre-commit-config.yaml`,
   a root `*.tf`) was actually touched.
4. If nothing was edited this session, say so and stop — don't run a full-repo
   pass by default. Ask the user if they want to scope it wider.

## Step 2 — Run `tofu fmt` (Terraform/OpenTofu directories only)

For each touched directory that contains `.tf` or `.tfvars` files:

```bash
tofu fmt <directory>
```

If `tofu` isn't on `PATH` in this shell, prefix with the homebrew path per this
workspace's tool conventions:

```bash
PATH="/opt/homebrew/bin:$PATH" tofu fmt <directory>
```

Run this per-directory, not recursively across the whole repo, to stay scoped.

## Step 3 — Run `pre-commit`

Run `pre-commit` scoped to the touched files, not `--all-files` (that would pull
in the whole repo):

```bash
PATH="/opt/homebrew/bin:$PATH" pre-commit run --files <file1> <file2> ...
```

Build the `<file1> <file2> ...` list from the same session-edited file set used in
Step 1 (not just the directories — pre-commit operates on files).

If pre-commit isn't installed as a git hook yet in this repo, that's fine —
`pre-commit run --files` works standalone without `pre-commit install`.

## Step 4 — Attempt Auto-Fixes

Some hooks (`yamlfmt`, `terraform_fmt`, `end-of-file-fixer`, `trailing-whitespace`,
`ruff-format`, `ruff-check --fix`) modify files in place and report "Failed - files
were modified by this hook." This is not a real failure — it's the hook doing its
job.

1. Re-run the same scoped `pre-commit run --files ...` command.
2. If it's clean on the second pass, done.
3. If a hook fails for a reason that isn't auto-fixable (e.g. `terraform_tflint`
   flagging a real issue, `gitleaks` finding a secret, `check-yaml` finding
   invalid syntax), read the failure output and fix the underlying issue directly:
   - tflint warnings: fix the flagged resource/variable
   - gitleaks hits: stop and flag to the user immediately — do not attempt to
     silently strip a real secret from a file without telling them; treat this
     as a security-sensitive finding per the workspace's safety guardrails
   - check-yaml / check-json: fix the malformed syntax at the reported line
4. Re-run after each fix until clean, or until you hit something that requires
   user input (ambiguous fix, secret found, missing tool).

## Step 5 — Report

Summarize concisely:
- Which directories/files were scoped
- What `tofu fmt` changed, if anything
- Which pre-commit hooks ran, which passed, which were auto-fixed, which needed
  manual fixes (and what you changed)
- Anything you could not fix and why (missing tool, needs user decision, secret
  found)

Don't dump full hook output back at the user — just the pass/fail summary and
what changed.

## Notes

- This skill does not run `tofu validate` or `tofu plan` — that requires provider
  registry access and state, out of scope for a formatting/lint pass.
- This skill does not stage or commit anything. It only formats/lints files in
  place. Committing is a separate, explicit step (see the `comment` skill for
  commit messages).
- If `tflint` or `terraform-docs` aren't on `PATH`, check `/opt/homebrew/bin`
  per this workspace's tool path conventions before reporting them as missing.
