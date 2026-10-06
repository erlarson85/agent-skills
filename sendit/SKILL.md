---
name: sendit
description: Commit staged/unstaged changes, push to a new branch, and open a PR in one go. Use when the user says "sendit", "ship it", or asks to commit, push, and open a PR together.
---

## Overview

`sendit` is the end-to-end wrap-up: take the current working tree changes, commit
them, push to a remote branch, and open a PR — in one flow instead of three
separate asks. It composes with the `tidy` skill (formatting/lint pass) and the
`comment` skill (commit/PR message generation) rather than duplicating them.

Invocation: `/sendit`

## Preconditions

1. Confirm there are actual changes to send: `git status --short`. If nothing is
   staged or modified and the branch is already ahead of origin only by unpushed
   commits, skip straight to Step 3 (push) and Step 4 (PR).
2. If nothing changed and nothing is unpushed, say so and stop.
3. Identify the repo root the user is working in. If the workspace has multiple
   repos open, use the one containing the active/open editor files, or ask if
   ambiguous.

## Step 1 — Branch Safety

Per workspace git-safety rules: never commit directly to `main`/`master`.

```bash
git branch --show-current
```

- If the current branch is `main` or `master`, create a new branch before
  committing. Derive a short, descriptive branch name from the changed files or
  the user's stated intent (kebab-case, e.g. `add-ansible-integration-cvd-vad-prod`).
  ```bash
  git checkout -b <descriptive-branch-name>
  ```
- If already on a feature branch, stay on it.

## Step 2 — Stage and Commit

1. Stage specific files by name — never `git add .` or `git add -A`:
   ```bash
   git add <file1> <file2> ...
   ```
   Build the file list from `git status --short`, limited to files relevant to
   the current task (not unrelated dirty files elsewhere in the tree).
2. Flag any file that looks like it holds secrets (`.env`, `credentials.json`,
   `*.pem`, `*.tfvars` with embedded secrets, etc.) before staging it. Ask the
   user to confirm if unsure.
3. Generate the commit message using the `comment` skill's `commit` mode
   (Conventional Commits format) based on the actual diff:
   ```bash
   git diff --cached
   ```
4. Commit:
   ```bash
   git commit -m "<type>[scope]: <description>"
   ```
   Never use `--no-verify` unless the user explicitly asks to skip hooks. If a
   pre-commit hook modifies files, re-stage and create a new commit — never
   `--amend` unless the user explicitly asks.

## Step 3 — Push

```bash
git push -u origin <branch-name>
```

- Always push to a new/feature branch, never force-push, never push directly to
  `main`/`master`.
- If the branch already has an upstream, a plain `git push` is fine.

## Step 4 — Open the PR

Use the `comment` skill's `pr` mode to generate the title and body from the
branch diff (`git diff main...HEAD`, or `master` if `main` doesn't exist) and the
commit log (`git log main..HEAD --oneline`).

```bash
gh pr create --title "<title>" --body "<body>"
```

- Title ≤ 70 characters, Conventional Commits style.
- Body includes a summary of changes and what was tested (per the `comment`
  skill's `pr` structure: Summary, Changed Files, Impact, Breaking Changes,
  Testing).
- Use `glab mr create` instead if the repo remote is GitLab, not GitHub.
- If an open PR already exists for this branch, report its URL instead of
  creating a duplicate.

## Step 5 — Report

Report concisely:
- Branch name created/used
- Commit SHA and message (one line)
- Push result
- PR URL

## Notes

- This skill does not run `tidy` automatically. If the user wants formatting/lint
  cleanup first, run `tidy` before `sendit`, or ask the user if they want both.
- This skill does not run tests or builds. If the project has a verification
  step the user cares about, run it before committing, per the workspace's
  general verification guideline.
- Terraform/OpenTofu changes: per workspace rules, this skill never runs `tofu`/
  `terraform` CLI commands (plan/apply/import/state) — those run exclusively
  through the Jenkins pipeline. `sendit` only handles git/PR mechanics; the
  pipeline picks up planning/apply after the PR is opened.
- If `gh` (or `glab`) is not authenticated or not installed, stop and tell the
  user rather than attempting a workaround.
