# Repository setup and layout

GitHub settings and repository shape required for the automation in [`ci.md`](./ci.md), [`lcm.md`](./lcm.md), and [`scanning.md`](./scanning.md).

## GitHub features to enable

| Feature | Why |
| --- | --- |
| GitHub Actions | CI, publish, refresh, automerge, Lane B |
| GitHub Packages (if GHCR profile or themes on GHCR/Packages) | Image and/or package publish/pull |
| Dependabot alerts | Lane A visibility |
| Dependabot security updates | Security PRs without waiting for weekly only |
| Dependabot version updates | From committed `.github/dependabot.yml` |
| Code scanning | Trivy SARIF under Security |
| Auto-merge | Dependabot patch/minor and Lane B crit/high digest PRs |
| Automatically delete head branches | Less branch clutter after Dependabot/bot merges |

## Actions permissions

Path: **Settings → Actions → General**

1. Actions enabled.
2. Workflow permissions: allow read and write permissions **or** job-level permissions that still allow packages/security as needed.
3. **Allow GitHub Actions to create and approve pull requests** — required for automerge approval and Lane B PR openers.
4. If actions are restricted, allow at least:
   - `actions/*`
   - `docker/*`
   - `dependabot/fetch-metadata`
   - `hmarr/auto-approve-action` (or successor used by the app)
   - `peter-evans/enable-pull-request-automerge`
   - `peter-evans/create-pull-request`
   - `aquasecurity/trivy-action` / `aquasecurity/setup-trivy`
   - `github/codeql-action/*`
   - any Artifactory/JFrog login or CLI actions you standardize on

## Pull request settings

Path: **Settings → General → Pull Requests**

- Allow auto-merge
- Automatically delete head branches

## Secrets and variables

| Secret | When required |
| --- | --- |
| `GITHUB_TOKEN` | Default; grant job permissions in YAML |
| Bot PAT / GitHub App token (e.g. `APP_BOT_TOKEN`) | Lane B or release-pin PRs that must trigger CI (token-from-GITHUB_TOKEN PRs often skip workflows) |
| Artifactory robot/identity (`ARTIFACTORY_USER` / `ARTIFACTORY_TOKEN` or org standard names) | WTG profile push/pull in CI |
| Themes feed auth | If `@mkronvold/themes` is private |
| `GHCR_TOKEN` on hosts | Private GHCR pulls when docker config is not enough |

Never commit tokens. Prefer host `~/.docker/config.json` for operators.

## Branch protection (`main` / default)

Require checks that match **stable job names** from the app’s workflows. Typical set:

1. Validate / workspace validate
2. Each service image build job (PR builds)
3. Each Trivy scan job (if separate)
4. Code scanning / new alerts policy as org allows

Automerge must wait on the same gates humans use.

Optional: require PR reviews for human changes; bot automerge still needs permission model that works with your org rules (sometimes a second bot approve or rulesets exceptions).

## GHCR package settings (home profile)

- Package visibility matches app intent (public/private).
- Deployment identities can pull.
- Link packages to the source repo when possible.

## Artifactory (WTG profile)

- CI robot can push to `sv4.art/repo.ops` (or org path).
- Runtime pull identities for Kubernetes/Docker hosts.
- Xray watch/policy attached to the repo/packages used for app images.
- Document the exact repository key and image path in the app’s `docs/CI.md` / `docs/LCM.md`.

## Recommended repository layout (product app)

```text
.
├── .github/
│   ├── dependabot.yml
│   └── workflows/
│       ├── publish-images.yml
│       ├── refresh-images.yml
│       ├── trivy-scan.yml          # or scan jobs inside publish
│       ├── automerge-dependencies.yml
│       ├── base-image-cve-fix.yml
│       └── automerge-base-image-cve.yml
├── apps/
│   ├── api/
│   ├── web/
│   └── worker/                    # optional
├── packages/
│   ├── schemas/ or contracts/
│   └── config/                    # optional
├── infra/
│   ├── docker/
│   │   ├── *.Dockerfile
│   │   ├── compose-prod.yaml
│   │   ├── compose-prod.images.env
│   │   ├── compose-prod.env.example
│   │   ├── up.sh
│   │   ├── down.sh
│   │   └── autoupdate.sh          # optional home channel
│   └── k8s/
│       └── <variants>/            # raw manifests / kustomize
├── security/
│   └── exceptions.md
├── .trivyignore.yaml
├── .xrayignore                    # when Artifactory/Xray in use
├── .node-version
├── package.json
├── pnpm-workspace.yaml
├── pnpm-lock.yaml
├── turbo.json
└── docs/
    ├── CI.md
    ├── LCM.md
    ├── DOCKER.md
    ├── GITHUB-SETUP.md
    └── KUBERNETES.md
```

Smaller profiles may collapse `infra/` or omit `worker/`, but product apps keep Compose pins + Kubernetes variants.

## App-local docs contract

Each maintained app should keep thin local docs that:

1. State registry profile (home vs WTG).
2. List actual workflow file names and image names.
3. Link to this repository for the standard.
4. Avoid forking policy text — describe only deltas.

## One-time setup checklist

- [ ] Features enabled (Actions, Dependabot, code scanning, auto-merge, delete branches)
- [ ] Actions can create/approve PRs
- [ ] Branch protection required checks named
- [ ] Secrets for bot token and registry profile
- [ ] `dependabot.yml` committed
- [ ] Workflows committed and green on a dry-run PR
- [ ] `security/exceptions.md` + ignore files present (even if empty templates)
- [ ] `docs/GITHUB-SETUP.md` points operators at this checklist
