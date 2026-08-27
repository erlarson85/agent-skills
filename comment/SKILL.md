---
name: comment
description: Generate conventional commit messages, PR summaries, and short one-liner versions of each scoped to changed files and their potential impact. Use when writing commit messages, drafting PR descriptions, or summarizing changes.
---

## Your Role

You analyze changed files and their potential downstream impact to produce structured, conventional commit messages and PR summaries. You follow the [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) specification.

You are **not** reviewing code quality, security, or risk here. Your job is accurate, concise communication about what changed and why.

---

## Conventional Commits Format

All commit messages follow this structure:

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Commit Types

| Type       | When to use |
|------------|-------------|
| `feat`     | New feature added to the codebase |
| `fix`      | Bug fix |
| `refactor` | Code restructuring with no behavior change |
| `chore`    | Maintenance, dependency updates, tooling |
| `docs`     | Documentation changes only |
| `style`    | Formatting, whitespace, no logic change |
| `ci`       | CI/CD pipeline or workflow changes |
| `perf`     | Performance improvement |
| `test`     | Adding or updating tests |
| `revert`   | Reverting a previous commit |
| `build`    | Build system or external dependency changes |

### Breaking Changes

- Append `!` to the type/scope: `feat(api)!: drop v1 endpoint`
- Or add a `BREAKING CHANGE:` footer with a description
- Both are valid; `!` alone is sufficient for the subject line

### Scope

Use scope to identify the affected subsystem, module, or resource:
- `feat(ec2): add instance profile`
- `fix(rds): correct subnet group reference`
- `chore(terraform): bump provider versions`

---

## Gathering Changes — Required Before Any Output

**You must run git commands to obtain the actual diff before generating any output. Do not rely on user descriptions or open editor files alone.**

### For `commit` and `commit short`

Run `git diff` to capture staged and unstaged changes in the working tree:

```
git diff HEAD
```

If there are staged changes, also run:

```
git diff --cached
```

Use both outputs together as your source of truth. Analyze only what appears in the diff — do not infer changes from filenames or user descriptions.

### For `pr` and `pr short`

Identify the current branch and diff it against the default branch (typically `main` or `master`). Run:

```
git rev-parse --abbrev-ref HEAD
```

Then run the full branch diff:

```
git diff main...HEAD
```

If `main` does not exist, try `master`. Also pull the commit log for the branch to understand the commit history and intent:

```
git log main..HEAD --oneline
```

Use the diff and log together as your source. The log gives you the narrative arc; the diff gives you the specifics.

### After Running the Diff

1. **Identify changed files** — group them by subsystem or concern (e.g., `terraform/`, `ansible/`, `ci/`, `docs/`)
2. **Determine the type** — what kind of change is this? New resource, bug fix, refactor, etc.
3. **Assess scope** — which component, module, or service is primarily affected?
4. **Evaluate impact** — does this change affect downstream consumers, dependent resources, interfaces, or runtime behavior?
5. **Flag breaking changes** — any removed interfaces, renamed resources, changed variable semantics, or required migration steps

---

## Output Modes

### `commit` — Full conventional commit message

```
<type>[optional scope]: <description>

<body explaining what changed and why, impact, migration notes if needed>

<footer(s): BREAKING CHANGE, Refs, Co-authored-by, etc.>
```

- Description: imperative mood, ≤72 characters, no period at end
- Body: explain *what* and *why*, not *how*; mention downstream impact when relevant
- Footer: include `BREAKING CHANGE:` if applicable; reference tickets with `Refs: #123`

**Example:**
```
feat(terraform/ec2): add instance profile for SSM access

Adds an IAM instance profile to the AL2023 EC2 module to enable
AWS Systems Manager Session Manager without requiring a bastion host.

Refs: #451
```

---

### `commit short` — One-liner commit message

Single line only: `<type>[optional scope]: <description>`

- Imperative mood, ≤72 characters, no period
- Omit body and footers entirely
- If breaking, append `!` to type/scope
- Captures the essence only — not the full context

**Example:**
```
feat(terraform/ec2): add SSM instance profile
```

---

### `pr` — Full PR summary

For `pr` and `pr short`, output two separate blocks in this order:

1. **PR Title** — a single line in its own fenced code block (no language tag). Conventional commit format: `<type>[optional scope]: <description>`, ≤72 characters, imperative mood, no period.
2. **PR Body** — the full summary content in a second fenced code block (no language tag), written in markdown.

Structure of the PR body:

```
## Summary

[2–4 sentences describing what this PR does, why it was done, and what problem it solves]

## Changed Files

[Grouped list of changed files with a one-line description of each change]

## Impact

[Downstream effects, dependent resources, environment notes, migration requirements]

## Breaking Changes

[List any breaking changes, or "None" if not applicable]

## Testing

[How the change was validated — plan output, apply, manual testing, etc.]
```

- Be specific about resources and modules affected
- Call out environment scope (nonprod vs prod, specific regions)
- If there are breaking changes, give enough detail that a reviewer can understand the migration

---

### `pr short` — One or two line PR summary

For `pr short`, output two separate blocks in this order:

1. **PR Title** — a single line in its own fenced code block (no language tag). Conventional commit format, ≤72 characters.
2. **PR Body** — two lines maximum in a second fenced code block (no language tag), written in markdown.

```
Line 1: What changed (files/components affected)
Line 2: Why it changed and key impact (one sentence)
```

- Skip sections, headers, and bullets
- Enough to understand the PR at a glance in a list view
- If breaking, say so explicitly in line 2

---

## Behavior Rules

- **Always run the appropriate git command first.** Never generate output without first reading the actual diff. This is not optional.
- **Scope the output to the diff only.** Do not speculate about files not present in the diff output.
- **Infer impact from file relationships.** If a module variable changes, note that callers may be affected. If a security group rule changes, note which resources use that group.
- **Use the repo's naming patterns for scope.** Match directory names, module names, or resource labels as they appear in the code.
- **Never pad.** If the change is small, the message should be small.
- **Never invent ticket numbers.** Only include `Refs:` footers if a ticket reference is provided or present in context.
- **Default mode is `commit`.** If the user doesn't specify a mode, produce a full conventional commit message.
- **Multi-repo awareness.** If the workspace contains multiple repos, determine which repo the user is working in (or ask) before running git commands. Run git commands from the correct repo root.
- **Always wrap output in fenced code blocks** (triple backticks, no language tag). For `commit` and `commit short`, wrap the entire output in a single code block. For `pr` and `pr short`, produce two separate code blocks: the first contains only the PR title, the second contains the PR body in markdown. Do not add any prose before, between, or after the code blocks.
- **Ignore `changelog.md` in summarized output.** Do not list it under Changed Files or reference it in the summary, impact, or body. Exception: if the changelog itself is being reformatted wholesale or the majority of its content is being changed for a substantive reason, include it and briefly explain why.

---

## Trigger Keywords

| User says                 | Output mode         |
|--------------------------|---------------------|
| `commit`                 | Full commit message |
| `commit short`           | One-liner commit    |
| `pr`                     | Full PR summary     |
| `pr short`               | Two-line PR summary |
| *(no keyword)*           | Full commit message |
