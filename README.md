# DDT Agent Skills Library

Shared Kiro skills for ASU Engineering teams.

## Installation

### Workspace (single project)

Symlink or copy the skills you need into your project's `.kiro/skills/` directory:

```bash
# Symlink all skills
ln -s /path/to/ddt-agent-skills-library/skills/* .kiro/skills/

# Or copy a specific skill
cp -r /path/to/ddt-agent-skills-library/skills/pr-review .kiro/skills/
```

### Global (all projects)

Copy skills to `~/.kiro/skills/` for availability across all workspaces:

```bash
cp -r /path/to/ddt-agent-skills-library/skills/pr-review ~/.kiro/skills/
```

## Available Skills

| Skill | Slash Command | Description |
|-------|---------------|-------------|
| [pr-review](pr-review/) | `/pr-review` | Review PRs for code quality, security, and ASU standards |
| [repo-scaffold](repo-scaffold/) | `/repo-scaffold` | Scaffold new repos using ASU Copier templates |
| [jenkins-pipeline](jenkins-pipeline/) | `/jenkins-pipeline` | Write Jenkinsfiles following ASU Jenkins PoP patterns |
| [security-review](security-review/) | `/security-review` | Focused security audit for IAM, networking, and auth changes |
| [read](read/) | `/read` | Read a GitHub repo's main/master branch by short name, no clone needed |
| [tidy](tidy/) | `/tidy` | Run tofu fmt + pre-commit scoped to this session's edited folders |

## Usage

Skills activate automatically when Kiro matches your request, or invoke directly:

```
> /pr-review focus on IAM policy changes
> /repo-scaffold new terraform project called my-service
> /jenkins-pipeline add a terraform plan stage
> /security-review check the IAM changes in this PR
```

## Custom Agents

If using a custom agent, add skills to your agent's `resources`:

```json
{
  "resources": [
    "skill://.kiro/skills/*/SKILL.md"
  ]
}
```
