# Copier Templates Reference

## Available Templates

| Template | Source | Use Case |
|----------|--------|----------|
| `dot-devops-examples-terraform` | `https://github.com/ASU/dot-devops-examples-terraform.git` | Terraform infrastructure projects |

## .copier-answers.yml Format

This file is auto-generated and tracks the template version:

```yaml
_commit: v1.1.15
_src_path: https://github.com/ASU/dot-devops-examples-terraform.git
administrative_contact: <asurite>
jenkins_pop: true
product_family_key: <team-prefix>
product_key: <product-short-name>
providers: []
repository_name: https://github.com/ASU/<repo-name>
technical_contact: <asurite>
workspaces:
  - test
```

## Template Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `product_family_key` | Team/product-family prefix | `ddt`, `aiml`, `ceng` |
| `product_key` | Short product name | `asl`, `llm`, `vpc` |
| `administrative_contact` | ASURite of admin | `ccumaran` |
| `technical_contact` | ASURite of tech lead | `mshure1` |
| `jenkins_pop` | Enable Jenkins PoP pipeline | `true` |
| `providers` | Terraform providers to include | `[aws, azurerm]` |
| `workspaces` | Terraform workspace names | `[test, prod]` |
| `repository_name` | Full GitHub URL | `https://github.com/ASU/ddt-agent-skills-library` |

## Naming Conventions

- **Repos:** `{product_family_key}-{descriptive-name}` (e.g., `ddt-agent-skills-library`)
- **Teams:** `{product_family_key}-{product-name}-{role}` where role is `approvers`, `writers`, or `readers`
- **S3 Bucket:** `asu-uto-ops-mgmt-terraform-state-jenkins-{product_family_key}`
- **State Key:** `https://github.com/ASU/{repo-name}/terraform/terraform.tfstate`
