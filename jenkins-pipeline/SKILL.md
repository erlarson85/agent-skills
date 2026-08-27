---
name: jenkins-pipeline
description: Write and troubleshoot Jenkinsfiles following ASU Jenkins PoP patterns. Use when creating pipelines, adding stages, fixing build failures, or configuring Jenkins for Terraform projects.
---

## Architecture

ASU runs two types of Jenkins:

- **PoP (Pipeline on Pod):** Kubernetes-based. Builds execute in pods with sidecar containers.
- **Non-PoP (Legacy):** EC2-based with Docker daemon.

The `terraform()` step handles both automatically — you do not need to detect which environment you're in.

The default IaC tool is **OpenTofu** (`tofu`), not `terraform`.

## Standard Jenkinsfile (PoP)

```groovy
#!groovy
def slack_channel = '#prdfam-TEAM-ci'

pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
  - name: terragrunt
    image: asuuto/dco-terragrunt:latest
    imagePullPolicy: Always
    command:
    - cat
    tty: true
  imagePullSecrets:
  - name: docker-hub-credentials
"""
    }
  }
  options {
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds()
  }
  triggers {
    cron('H H * * *')
  }
  stages {
    stage('Terraform Non-Prod') {
      steps {
        container('terragrunt') {
          script {
            terraform(
              workspace: 'dev',
              vault_callback: { opsVaultLogin() },
            )
          }
        }
      }
    }
    stage('Terraform Prod') {
      steps {
        container('terragrunt') {
          script {
            terraform(
              workspace: 'prod',
              vault_callback: { opsVaultLogin() },
            )
          }
        }
      }
    }
  }
  post {
    always {
      slackNotification(slack_channel)
    }
  }
}
```

## Standard Jenkinsfile (Non-PoP / Legacy)

```groovy
#!groovy
def slack_channel = '#prdfam-TEAM-ci'

pipeline {
  agent {
    label 'ec2-docker'
  }
  options {
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds()
  }
  triggers {
    cron('H H * * *')
  }
  stages {
    stage('Terraform Non-Prod') {
      steps {
        script {
          terraform(
            workspace: 'np',
            apply_on_branch: '*',
            vault_callback: { opsVaultLogin() },
          )
        }
      }
    }
    stage('Terraform Prod') {
      steps {
        script {
          terraform(
            workspace: 'prod',
            vault_callback: { opsVaultLogin() },
          )
        }
      }
    }
  }
  post {
    always {
      slackNotification(slack_channel)
      clean()
    }
  }
}
```

## The `terraform()` Step

This is the primary step for all Terraform/OpenTofu workloads. It wraps `terraformCore` and automatically handles PoP vs non-PoP container execution.

```groovy
terraform(
  workspace: 'prod',                 // terraform workspace
  vault_callback: { opsVaultLogin() }, // vault auth closure
  apply_on_branch: 'main',           // only apply on this branch (* = all)
  auto_approve: false,               // skip approval gate
  auto_approve_non_destructive: false, // auto-approve if no destroys
  override_freeze: false,            // bypass change freeze (USE WITH CAUTION)
  maintenance_mode: false,           // bypass lint on non-protected branches
  tool: 'tofu',                      // tofu or terraform
  variant: '',                       // 'tofu-python' for Python workloads
  apply_submitters: '',              // restrict who can approve
)
```

## Shared Pipeline Library

All repos have access to `devops-jenkins-pipeline-library`. Key steps:

| Step | Purpose |
|------|---------|
| `terraform` | Full plan/apply lifecycle (auto-detects PoP vs non-PoP) |
| `terraformCore` | Lower-level plan/apply (use `terraform()` instead) |
| `changeFreezeCheck` | Block deploys during change freeze |
| `scanDockerImage` / `scanContainerImage` | Grype vulnerability scanning |
| `vaultLogin` / `opsVaultLogin` | HashiCorp Vault authentication |
| `getArtifactoryCredentials` | JFrog creds from Vault |
| `setup{Pip,Npm,Uv,Poetry,Gradle,Maven}Credentials` | Package manager auth |
| `ansible` / `ansibleKubernetes` | Ansible execution (non-PoP / PoP) |
| `slackNotification` | Slack failure alerts |
| `servicenow_change` | ServiceNow change requests |
| `gitAutoVersionTag` | Semantic versioning |
| `isJenkinsPop` | Detect PoP vs non-PoP |
| `clean` | Workspace cleanup (non-PoP only) |

## Credentials

- **AWS:** Provided via IRSA (IAM Roles for Service Accounts). Never use `withCredentials` for AWS.
- **Vault:** Use `vault_callback` closure pattern for short-lived tokens.
- **GitHub:** Handled automatically by `terraform()` (`github-org-asu-pac` on PoP, `github-pat-asu-jenkins-devops` on non-PoP).
- **Docker Hub:** Pull secret `docker-hub-credentials` in pod spec.
- **Artifactory:** `getArtifactoryCredentials()` fetches from Vault.

## CPS Groovy Pitfalls

- No `?:` (elvis) in `for` expressions — assign to variable first
- All local variables must be Serializable (no `JsonSlurper`)
- `@NonCPS` skips CPS transform at runtime but parser restrictions still apply

## PR Integration

- `tfcmt` posts plan/apply results as PR comments automatically
- Labels added on failure: `terraform/fmt`, `terraform/tflint`, `maintenance-mode-enabled`
- Deprecated module checks fail the build (git-sourced modules, security-group module)

## Post Block

- **PoP:** Use `slackNotification(slack_channel)` — no `clean()` needed (pods are ephemeral)
- **Non-PoP:** Use both `slackNotification(slack_channel)` and `clean()`

## Troubleshooting

See `references/jenkinsfile-patterns.md` for advanced patterns, multi-workspace deploys, approval gates, and common errors.
