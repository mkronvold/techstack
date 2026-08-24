# Repo Actions Hub Canvas

Enable the GitHub Actions workflow management sidepanel in GitHub Copilot CLI sessions.

## Overview

The **Repo Actions Hub** is a built-in canvas that provides a convenient sidepanel interface to:
- Browse GitHub Actions workflows in your repository
- Inspect recent workflow runs and their results
- Trigger `workflow_dispatch` enabled workflows directly from the session

This guide explains how to enable it on any GitHub-backed project.

## Setup

### 1. Create `.github/github-app.yml`

Add this configuration file to your repository:

```yaml
canvases:
  - id: repo-actions-hub
    title: Repo Actions
    description: View and manage GitHub Actions workflows
    icon: workflow
```

**Location:** `.github/github-app.yml` (in repository root)

### 2. Commit and push to main

The configuration is scoped to the repository and will take effect for all new sessions created from that repository.

```bash
git add .github/github-app.yml
git commit -m "feat: Enable Repo Actions Hub canvas sidepanel"
git push origin main
```

## Usage

Once configured:

1. Create or open a session from your repository
2. The **Repo Actions** tab will appear in the Copilot CLI sidepanel
3. Select any workflow to see:
   - Recent run history
   - Run status and logs
   - Trigger options for `workflow_dispatch` workflows
4. Click "Run" to trigger a workflow directly from the session

## Benefits

- **Quick workflow access** — no need to switch to GitHub web
- **Session-aware** — context is preserved while working on code
- **Dispatch workflows** — trigger CI/CD from the CLI during development
- **Monitoring** — check deployment and test runs without leaving your session

## Customization (Optional)

The canvas uses sensible defaults, but you can customize the sidepanel title and icon:

```yaml
canvases:
  - id: repo-actions-hub
    title: "GitHub Workflows"  # Custom title
    description: "Manage and monitor CI/CD workflows"
    icon: gear  # Alternative icons: workflow, gear, play, etc.
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Canvas doesn't appear | Ensure `.github/github-app.yml` is committed to `main` branch |
| Can't trigger workflows | Verify the workflow has `workflow_dispatch` trigger enabled |
| Permissions error | Ensure your GitHub token has `actions: read/write` permissions |

## See Also

- [GitHub Copilot App Reference](https://docs.github.com/copilot/reference/github-copilot-app-reference/repository-configuration)
- [GitHub Actions Workflows](https://docs.github.com/en/actions/workflows)
