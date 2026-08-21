# techstack

Shared technology and operations baseline for multi-app maintenance with low or no ongoing touch.

Use this repository when starting or remediate-standardizing an app so tooling, CI, scanning, lifecycle, and GitHub setup stay consistent across projects.

## Documents

| Doc | Covers |
| --- | --- |
| [`techstack.md`](./techstack.md) | Application profiles, tooling decision matrix, repo layout, Node/package defaults, themes package consumption |
| [`image-policy.md`](./image-policy.md) | Upstream-first image terminology, derived application-image standard, controls, exceptions, and TSX/Node/Postgres patterns |
| [`ci.md`](./ci.md) | Required validation, image build/publish, themes at build time, registry profiles (GHCR vs Artifactory), evidence |
| [`lcm.md`](./lcm.md) | Image refresh cadence, immutable release pins, optional home auto-update, workload impact and how to avoid churn |
| [`templates/compose-autoupdate/`](./templates/compose-autoupdate/) | Vendored canonical Compose auto-update script, configuration, systemd user timer, and portable safety tests |
| [`scanning.md`](./scanning.md) | Dependabot lanes, Trivy, Xray, exception registries, Lane B digest automation |
| [`repo-setup.md`](./repo-setup.md) | GitHub repository settings, branch protection, layout checklist |
| [`remediation.md`](./remediation.md) | Per-app alignment, session kickoffs, priority IDs (all apps; not split by home/WTG) |

## Locked defaults (v1)

| Topic | Default |
| --- | --- |
| Product-app package manager | Corepack `pnpm@10.x` + Turbo |
| Node | One declared major (forward target **26**) everywhere: engines, CI, Dockerfiles |
| Themes | Depend on published `@mkronvold/themes` (lockfile + Dependabot); do not curl `main` at build time |
| Registry profile | **Home:** `ghcr-dev` on GHCR. **WTG/org:** protected `sv4.art/repo.ops` publication with `repo.ops` as the only runtime pull host; GHCR optional mirror |
| Scanning | Trivy High/Critical fail + git exceptions everywhere; Xray additional gate on Artifactory publishes |
| Exceptions | Machine file (`.trivyignore.yaml` and/or `.xrayignore`) **and** human `security/exceptions.md` |
| Deploy channels | **Required:** immutable `tag@sha256` release pins for Compose **and** raw Kubernetes. **Optional:** home mutable tag + `autoupdate.sh` |
| Lane B | Required for maintained container apps (base digest-fix PRs; crit/high auto-merge when green) |
| Workflows in this repo | Docs-only v1 — implement in each app; reference `mkronvold/revu` and `mkronvold-wtg/tavi` |

## How to use on a new or existing app

1. Classify the app with [`techstack.md`](./techstack.md) (profile + layout + core tech).
2. Apply [`repo-setup.md`](./repo-setup.md) settings and layout.
3. Implement CI/publish per [`ci.md`](./ci.md) for the correct registry profile.
4. Wire scanning and Dependabot per [`scanning.md`](./scanning.md).
5. Document and automate lifecycle per [`lcm.md`](./lcm.md).
6. Track remaining gaps with a short app-local `docs/` pointer back here.

## Reference implementations

| Concern | Stronger reference today |
| --- | --- |
| Lane B digest bot, host `autoupdate.sh`, LCM operator polish | [`mkronvold/revu`](https://github.com/mkronvold/revu) |
| pnpm/Turbo product shape, release-pin PR, K8s pins, fail-closed Trivy | [`mkronvold-wtg/tavi`](https://github.com/mkronvold-wtg/tavi) |
| Shared visual tokens | [`mkronvold/themes`](https://github.com/mkronvold/themes) (must become a published package — see remediation) |

## Non-goals (v1)

- Shipping reusable called workflows from this repo
- Replacing app-specific domain docs (auth, backups, product UX)
- Forcing every tiny standalone tool into a full product-app workspace
