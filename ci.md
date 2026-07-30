# Continuous integration and image publish

PR/release validation, container build/publish, themes at build time, and registry profiles. Lifecycle promotion and host deploy live in [`lcm.md`](./lcm.md). Scanner policy lives in [`scanning.md`](./scanning.md).

## Goals

1. Every PR gets the same deterministic checks a human would run locally.
2. Every deployable service builds a container on PR (no push) and publishes outside PRs.
3. Published artifacts carry evidence (SBOM/provenance) and are scanned.
4. Registry choice follows the **home vs WTG/org** profile without forking the rest of the pipeline.
5. Themes come from the lockfile-backed package, not an ad-hoc network fetch.

## Required local/CI commands (product app)

| Step | Command shape | Notes |
| --- | --- | --- |
| Install | `corepack pnpm install --frozen-lockfile` | `npm ci` only for legacy single-ecosystem npm apps |
| Generate | Prisma client / package builds as needed | Fail if generated artifacts are required at test time |
| Lint | `pnpm lint` | |
| Typecheck | `pnpm typecheck` | |
| Test | `pnpm test` | Include DB service in CI when tests need it |
| Build | `pnpm build` | Production builds for all publishable packages/apps |
| Format check | `pnpm format:check` or equivalent | Optional separate job if folded into validate script |

Expose a single `validate` (or CI job equivalent) that runs the full gate in order.

Tiny standalone apps may reduce to `check` + deployable `build`.

## Required workflows (implement per app)

Docs-only standard: copy/adapt from reference repos; do not call reusable workflows from this repo in v1.

| Workflow | When | Must do |
| --- | --- | --- |
| `publish-images` (name may vary) | PR, push to default branch, `v*` tags, `workflow_dispatch` | Validate → matrix build each service image → scan → publish when not a PR |
| `refresh-images` | Weekly schedule + manual | Same validate/build/scan/publish for candidate `latest` / `refresh-*` tags; **do not** mutate `sha-*` tags |
| `trivy-scan` or scan steps inside publish | PR + default branch + after refresh | Per-service Trivy; SARIF categories; fail High/Critical subject to exceptions — see [`scanning.md`](./scanning.md) |
| `automerge-dependencies` | Dependabot PR events | Auto-approve/merge eligible patch/minor only |
| `base-image-cve-fix` + `automerge-base-image-cve` | After successful publish/refresh, schedule, manual | **Required** Lane B digest PRs — see [`scanning.md`](./scanning.md) |
| Release-pin job | On `v*` publish success | Open PR updating Compose **and** raw Kubernetes image refs to `tag@sha256:digest` |

### Reference implementations

| Workflow family | Prefer looking at |
| --- | --- |
| Validate + GHCR publish + inline Trivy/SBOM + Lane B | [`mkronvold/revu`](https://github.com/mkronvold/revu) `.github/workflows/` |
| pnpm validate, path filters, provenance, CycloneDX artifacts, release-pin PR, fail-closed Trivy | [`mkronvold-wtg/tavi`](https://github.com/mkronvold-wtg/tavi) `.github/workflows/` |

## Image build rules

| Rule | Detail |
| --- | --- |
| Context | Repository root (or documented monorepo context); one Dockerfile per service |
| Pins | Base images `tag@sha256:digest` |
| PR builds | `push: false`, load or build for scan; required check names stable for branch protection |
| Non-PR publish | Push tags: `latest` (default branch), `sha-*` or `type=sha`, branch, and version tags on `v*` |
| Refresh publish | `latest` + `refresh-YYYYMMDD-HHMMSS` only; never rewrite immutable sha tags for the same commit |
| Evidence | BuildKit SBOM + provenance on publish; retain CycloneDX and/or Trivy JSON artifacts (recommend ≥ 90–180 days) |
| Matrix | One job per deployable service (`api`, `web`, `worker`, …) with distinct scan category |
| Cache | GHA cache scoped per image is fine; refresh may use `no-cache: true` + `pull: true` |

## Themes at build time

| Requirement | Detail |
| --- | --- |
| Dependency | App `package.json` depends on `@mkronvold/themes` (or successor published name) |
| Install path | Same frozen lockfile install as the rest of the app — **no** `git clone`/`curl` of themes in Dockerfile or CI unless bootstrapping the package itself |
| Docker | Builder stage runs package manager install so themes resolve from the lockfile/registry config |
| Registry auth | If themes is on GitHub Packages or a private feed, configure CI/Docker auth via secrets; never bake tokens into layers |
| Verification | Unit/build failure if CSS/tokens import breaks; optional `pnpm why @mkronvold/themes` in docs for operators |
| Dependabot | npm/pnpm ecosystem entry covers themes version bumps with other deps |

## Registry profiles

Choose **one primary** profile per deployment audience. Pipelines stay the same; login/push destinations differ.

### Home profile

| Item | Value |
| --- | --- |
| Primary registry | `ghcr.io` |
| Image name | `ghcr.io/<owner>/<app>-<service>` |
| Auth (CI) | `GITHUB_TOKEN` packages:write |
| Auth (hosts) | `docker login ghcr.io` or `GHCR_USERNAME` / `GHCR_TOKEN` |
| Xray gate | Not required |
| Typical consumers | Home Docker hosts, public/private GHCR pulls |

### WTG / org profile

| Item | Value |
| --- | --- |
| Primary registry | Artifactory Docker repo `sv4.art/repo.ops` (confirm exact repository path per org standards) |
| Image name | `sv4.art/repo.ops/<app>-<service>` (or org-standard path) |
| Optional mirror | Also push the same tags/digests to GHCR when useful for home or backup pulls |
| Auth (CI) | Artifactory identity token / robot user via Actions secrets; never in git |
| Auth (runtime) | Cluster pull secrets / host docker config for Artifactory |
| Xray gate | **Required** on the published Artifactory digest — see [`scanning.md`](./scanning.md) |
| Typical consumers | Org Kubernetes and org Docker hosts |

### Dual-push shape (when mirror enabled)

1. Build once.
2. Tag for primary registry (and optional GHCR).
3. Push primary; record digest.
4. Push mirror with the same digest/tags when configured.
5. Scan the digest that will actually run (at minimum the primary).
6. Release-pin PRs reference the **primary** registry string for that app’s deploy profile.

### Profile decision

| If | Then |
| --- | --- |
| Only self-hosted home operators | Home / GHCR |
| Org/WTG runtime is source of truth | WTG / Artifactory primary |
| Both audiences actively deploy | WTG primary + GHCR mirror, **or** two pin files/docs clearly separated by channel |

## Tag and promotion contract

| Tag / ref | Meaning | May deploy automatically? |
| --- | --- | --- |
| `sha-*` / immutable digest | Exact build | Only via release-pin or explicit pin |
| `vX.Y.Z` | Release publish | After release-pin PR merge |
| `latest` | Candidate / mutable home channel | Home `autoupdate.sh` only if opted in |
| `refresh-*` | Diagnostic rebuild evidence | No production immutable channel |

Publishing is **not** deploying. See [`lcm.md`](./lcm.md).

## CI permissions (workflow tokens)

Minimum patterns (exact jobs vary):

| Job | Typical permissions |
| --- | --- |
| Validate | `contents: read` |
| Build/push GHCR | `packages: write`, `id-token: write`, `attestations: write` |
| SARIF upload | `security-events: write` |
| Release-pin PR | `contents: write`, `pull-requests: write` |
| Artifactory push | Registry secrets; no special GitHub perm beyond checkout |

Repository settings that must allow these flows are in [`repo-setup.md`](./repo-setup.md).

## Node and package manager matrix

| Surface | Product app default | Legacy npm app |
| --- | --- | --- |
| Node | 26.x from `.node-version` + `engines` | Same major rule |
| Package manager | Corepack pnpm 10.x | npm with `npm ci` |
| Lockfile | `pnpm-lock.yaml` | `package-lock.json` |

## Definition of done for CI on an app

- [ ] Validate job green on every PR
- [ ] Each service image builds on every PR
- [ ] Publish on default branch and tags to the correct registry profile
- [ ] SBOM/provenance (or documented equivalent) on publish
- [ ] Trivy per service with SARIF + fail High/Critical policy
- [ ] Weekly refresh workflow
- [ ] `v*` opens release-pin PR for Compose + Kubernetes
- [ ] Themes resolved only via package manager / lockfile
- [ ] Docs link here and describe any app-specific matrix entries
