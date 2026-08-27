---
name: repo-scaffold
description: Scaffold new GitHub repositories using ASU Copier templates and conventions. Use when creating new projects, setting up repo structure, or initializing Terraform repos with ASU standards.
---

## Overview

ASU uses [Copier](https://copier.readthedocs.io/) templates to scaffold new repositories with standard structure. The primary template for Terraform projects is `dot-devops-examples-terraform`.

## Scaffold a New Terraform Repo

1. **Create the repo** in the ASU org following naming: `{product-family}-{product-name}`
2. **Run Copier** from the template:

```bash
copier copy https://github.com/ASU/dot-devops-examples-terraform.git .
```

3. **Answer the prompts** (these populate `.copier-answers.yml`):
   - `product_family_key`: team prefix (e.g., `ddt`, `aiml`, `ceng`)
   - `product_key`: short product identifier
   - `administrative_contact`: ASURite of admin contact
   - `technical_contact`: ASURite of tech contact
   - `jenkins_pop`: true/false for Jenkins PoP integration
   - `workspaces`: list of Terraform workspaces (e.g., `[test, prod]`)

## Generated Structure

```
.
├── .copier-answers.yml
├── .github/
│   └── workflows/
│       └── pre-commit.yml
├── .gitignore
├── .pre-commit-config.yaml
├── .yamlfmt.yaml
├── AGENTS.md
├── CODEOWNERS
├── Jenkinsfile
├── README.md
└── terraform/
    ├── main.tf
    ├── module-product-tags.tf
    ├── outputs.tf
    ├── providers.tf
    ├── terraform.config
    ├── terraform.tf
    ├── test.tfvars
    └── variables.tf
```

## CODEOWNERS Pattern

```
*                       @ASU/{product-family}-{product-name}-approvers
CODEOWNERS              @ASU/devops-admins
/.github/               @ASU/devops-infrastructure
.copier-answers.yml     @ASU/devops-infrastructure
Jenkinsfile             @ASU/devops-infrastructure
.pre-commit-config.yaml @ASU/devops-infrastructure
.yamlfmt.yaml           @ASU/devops-infrastructure
/terraform/             @ASU/devops-infrastructure
```

## Pre-commit Hooks (Standard)

All repos should include these hooks. See `references/copier-templates.md` for the full config.

- `conventional-pre-commit` — enforce conventional commits
- `end-of-file-fixer`, `trailing-whitespace`, `mixed-line-ending`
- `check-json`, `check-yaml`, `pretty-format-json`
- `terraform_fmt`, `terraform_docs`, `terraform_tflint`
- `yamlfmt`
- `check-github-workflows`

## Updating an Existing Repo

To pull in template updates:

```bash
copier update
```

This respects `.copier-answers.yml` and merges changes from the upstream template.

## Post-Scaffold Checklist

- [ ] Create GitHub teams: `{prefix}-{name}-approvers` and `{prefix}-{name}-writers`
- [ ] Configure branch protection on `main`
- [ ] Verify Jenkins PoP webhook is configured
- [ ] Run `pre-commit install` locally
- [ ] Commit initial scaffold with message `feat: initial repository scaffold`
