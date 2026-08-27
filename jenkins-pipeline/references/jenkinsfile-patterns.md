# Jenkinsfile Patterns & Troubleshooting

## Multi-Workspace Deploy

Deploy to multiple workspaces with `terraformCore`:

```groovy
stage('Terraform') {
  steps {
    container('terragrunt') {
      script {
        def workspaces = ['sandbox', 'dev', 'qa', 'prod']
        for (ws in workspaces) {
          terraformCore(
            workspace: ws,
            tool: 'tofu',
            apply_on_branch: 'main'
          )
        }
      }
    }
  }
}
```

## Single-Environment Pipeline (Legacy Non-PoP)

The library provides `pipelineTerraformSingleEnvironment` for simple use cases:

```groovy
pipelineTerraformSingleEnvironment('#my-slack-channel', 'terraform')
```

This handles validate → plan → approve → apply with Slack notifications.

## Vault Integration

```groovy
terraformCore(
  workspace: 'prod',
  tool: 'tofu',
  vault_callback: {
    def (addr, token) = opsVaultLogin()
    return [addr, token]
  }
)
```

The callback is invoked twice (once for plan, once for apply) to keep token TTL short.

## Approval Gate Customization

```groovy
terraformCore(
  workspace: 'prod',
  tool: 'tofu',
  auto_approve: false,
  auto_approve_non_destructive: true,  // skip approval if no destroys
  apply_submitters: 'admin1@asu.edu,admin2@asu.edu'
)
```

## Change Freeze

`changeFreezeCheck()` is called automatically by `terraformCore`. To override:

```groovy
terraformCore(
  workspace: 'prod',
  override_freeze: true  // USE WITH CAUTION
)
```

## ServiceNow Integration

```groovy
terraformCore(
  workspace: 'prod',
  create_service_now_change_request: true,
  service_now_change_request_params: [
    short_description: "Deploy infra changes",
    // additional params
  ]
)
```

## Docker Image Scanning

```groovy
stage('Scan') {
  steps {
    container('terragrunt') {
      script {
        scanContainerImage(
          image: "my-image:latest",
          fail_on: "high"
        )
      }
    }
  }
}
```

## Daily Drift Detection

```groovy
triggers {
  cron('@daily')
}
```

With `terraformCore`, plan runs on schedule. If changes are detected on `main`, it posts to Slack.

## Common Failures

| Error | Cause | Fix |
|-------|-------|-----|
| `terraform/fmt` label on PR | Code not formatted | Run `tofu fmt -recursive` locally |
| `terraform/tflint` label on PR | Lint failures | Run `tflint --var-file=<ws>.tfvars` |
| `Terraform modules from Git are deprecated` | Using `git::ssh://` source | Switch to Artifactory module source |
| `security-group module is deprecated` | Using old security-group module | Use `aws_security_group` resource directly |
| `container not found: terragrunt` | Container name mismatch | Ensure `container('terragrunt')` matches pod spec |
| `docker-hub-credentials not found` | Secret missing from namespace | Verify Jenkins namespace has pull secret |
| CPS serialization error | Non-serializable local var | Move to `@NonCPS` method returning serializable types |
| `?:` parse error in for loop | Elvis in for expression | Assign to variable before for loop |

## Environment Variables

Automatically available in Jenkins PoP:

| Variable | Description |
|----------|-------------|
| `BRANCH_NAME` | Current git branch |
| `CHANGE_ID` | PR number (only on PR builds) |
| `BUILD_NUMBER` | Incrementing build number |
| `GIT_URL` | Repository URL |
| `GIT_COMMIT` | Current commit SHA |
| `JENKINS_URL` | Jenkins instance URL |

## Maintenance Mode

For troubleshooting failing checks on non-protected branches:

```groovy
terraformCore(
  workspace: 'dev',
  maintenance_mode: true  // bypasses lint on non-main branches ONLY
)
```

**Warning:** Merging to `main`/`master`/`prod` with failing checks still fails the build. Maintenance mode cannot bypass protected branch checks.
