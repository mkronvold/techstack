# Lifecycle management

How images stay fresh, how deployments consume them, and how to avoid surprising workload disruption. CI publish mechanics: [`ci.md`](./ci.md). Scanner-driven digest fixes: [`scanning.md`](./scanning.md).

## Model (two paths, do not collapse them)

| Path | What it updates | Source of truth |
| --- | --- | --- |
| **Image refresh automation** | Rebuild/republish containers from **current git pins** so OS layers and Dockerfile install steps pick up fixes inside those pins | Git pins + scheduled `refresh-images` |
| **Dependency / pin automation** | Manifests and lockfiles: npm/pnpm, Actions, Dockerfile/Compose/K8s digests | Dependabot + Lane B digest bot + human majors |

Refreshing without bumping pins does **not** float digest-pinned bases to a newer upstream digest. Pin bumps land first; refresh/publish then rebuild.

## Required deploy channel: immutable release pins

Every product app **must** support reviewed immutable deploys.

| Requirement | Detail |
| --- | --- |
| Image reference form | `registry/name:tag@sha256:digest` |
| Compose | Checked-in images env or equivalent (e.g. `compose-prod.images.env`) updated only by reviewable PR |
| Kubernetes | Raw manifests or kustomize variants for each supported layout; same digests as Compose for a given release |
| Promotion trigger | Successful `v*` image publish opens a **release-pin pull request**; humans merge to promote |
| `imagePullPolicy` | Prefer `IfNotPresent` with digest pins |
| Candidates | `latest` / `refresh-*` never silently rewrite immutable pins |
| Rollback | Revert the release-pin commit (or restore previous pin file values), re-apply, rollout status |

### Release-pin PR must update

1. Compose production image references for every app service.
2. Every supported Kubernetes deployment variant’s app container images.
3. Nothing else unrelated (no drive-by refactors).

### Operator apply (shape)

```bash
# Compose (example)
docker compose \
  --env-file infra/docker/compose-prod.images.env \
  --env-file infra/docker/compose-prod.env \
  -f infra/docker/compose-prod.yaml \
  pull
docker compose \
  --env-file infra/docker/compose-prod.images.env \
  --env-file infra/docker/compose-prod.env \
  -f infra/docker/compose-prod.yaml \
  up -d

# Kubernetes (example)
kubectl apply -k infra/k8s/<variant>
kubectl rollout status deployment/<app>-api -n <ns>
# …web, worker, etc.
```

## Optional channel: home / operator mutable auto-refresh

Trusted self-hosted Docker Compose instances **may** track a mutable app tag (usually `latest`) and self-update. This is **optional** and must not replace immutable pins for cluster or reviewed multi-env paths.

### When to enable

| Enable when | Avoid when |
| --- | --- |
| Single-operator or home lab wants low touch | Multi-env promotion, auditability, or shared prod cluster |
| App services are stateless enough that rolling restart via `up.sh` is safe | You cannot accept unattended restarts |
| Migrations are safe to run on every `up.sh` | Destructive migrate or manual gate required before migrate |

### Required `autoupdate.sh` behavior

| Requirement | Guidance |
| --- | --- |
| Location | Beside `up.sh` / `down.sh` and the production Compose file |
| Modes | Long-running interval **and** `--once` for cron/systemd |
| Detection | Resolve images from `docker compose config`; compare local digest to remote manifest (`Docker-Content-Digest` or registry API) |
| Registry | Match the app’s profile (`ghcr.io` for `ghcr-dev`; `repo.ops` only for Artifactory runtime pulls) |
| Scope | **Only** app images (`api` / `web` / `worker`). Never auto-refresh Postgres, proxies, or data-plane sidecars on the app cadence |
| Apply | `docker compose pull <changed>` then restart **only** through `./up.sh` |
| Startup path | Migrations, env reconcile, bootstrap live in `up.sh`; auto-refresh must reuse them |
| No-op exit | Distinct exit code (e.g. `10`) when no updates |
| Concurrency | `flock` (or equivalent); fail closed if locked |
| Secrets | Host docker config or env credentials; never print tokens; never commit tokens |
| Observability | Timestamped logs; post-update health checks; record deployed digests |
| Scheduling | Prefer `--once` every ~30 minutes per stack instance; one timer/log/lock per stack |

Do **not** use Watchtower unless an explicit exception is recorded.

### Canonical reusable template contract

Use the versioned, vendored
[`templates/compose-autoupdate/`](./templates/compose-autoupdate/) template as
the starting point. An app copies it under its reviewed `infra/docker/` tree
and brings later changes in through a sync PR; a host never downloads script
or application code at update time.

| Contract | Acceptance criterion |
| --- | --- |
| Required configuration | Working directory, Compose files, Compose env files, explicit service-to-image allowlist, registry profile, target platform, app `up.sh`/health/rollback commands, lock path, digest record path, and rollback image prefix are set and validated before Docker is called |
| Safety scope | The rendered Compose image for every allowed service matches its allowlist entry; only those services are passed to `docker compose pull` and app commands; DBs, proxies, migration jobs, caches, and sidecars are never listed or operated |
| Remote comparison | The running image's registry manifest digest is compared to the remote tag digest; Docker resolves the configured target platform, while the Artifactory profile independently verifies its Linux/amd64 child manifest; registry/auth/manifest errors and missing digests fail closed |
| Modes and locking | `--once`, `--dry-run`, and optional interval operation exist; `flock -n` rejects a concurrent run; a confirmed no-op is logged and exits `10` |
| Apply and health | Changed services only are pulled, then restarted through the app's `up.sh --services ...`; health runs after restart; `--dry-run` makes no pull, restart, tag, or record mutation |
| Rollback | Before pull, retain every currently running image under a local rollback tag; on pull, start, or health failure restore those tags and invoke the app rollback path for only the changed services |
| Credentials and observability | Use Docker's configured credential store; never run `docker login`, echo configuration, or print tokens; atomically record before/after/remote digests after a healthy update |
| Tests | Portable shell tests use fake Docker and flock commands and cover configuration/argument errors, immutable pins, allowlist mismatch, no-op, dry-run, rollback, and concurrent-run rejection without a daemon or network |

`up.sh` and the configured health/rollback executables must accept
`--services <service>...` and `AUTOUPDATE_SERVICES`. During
`AUTOUPDATE_ROLLBACK=1`, `up.sh` must skip pulling and recreate only those
services from the local tags restored by the updater.

### Mutable development versus immutable release pins

The updater is for an explicitly enabled mutable development/home candidate
tag; it cannot rewrite, consume, or replace immutable `tag@sha256:digest`
release pins. `ghcr-dev` accepts only
`ghcr.io/<namespace>/<image>:latest`. `artifactory-repo-ops` accepts only
`repo.ops` runtime image references and requires a protected
source-to-pull digest mapping, `linux/amd64`, and matching
`org.opencontainers.image.revision`; `sv4.art` is publication provenance,
never a runtime pull endpoint. Required release-pin Compose and Kubernetes
deployments continue to use immutable digests and reviewed promotion PRs.

### Companion scripts

| Artifact | Role |
| --- | --- |
| `up.sh` | Canonical start/restart: env, pull, migrate/bootstrap, `compose up -d` |
| `down.sh` | Canonical stop |
| Compose prod | Services, health checks, networks, image refs |
| Runtime env file | Host secrets/URLs only — not the immutable pin source of truth |
| App `docs/LCM.md` / `docs/DOCKER.md` | Cadence, tags consumed, timer setup, rollback |

Reference polish: `mkronvold/revu` LCM + DOCKER docs and scripts. Tavi also ships `infra/docker/autoupdate.sh` with dual channel support.

## Cadence

```text
Weekly (batch):
  • Dependabot non-major groups (npm/pnpm, Actions)
  • Docker/K8s pin PRs as configured
  • scheduled image refresh + SBOM/scan
  • triage open medium/low digest PRs and alerts

Event-driven (do not wait for Monday):
  • Lane A: security/fix PRs for crit/high dependencies
  • Lane B: base-image digest-fix PRs (crit/high auto-merge when green)
  • v* release publish → release-pin PR
```

## Workload impact and how to avoid pain

Unattended image movement **will** restart containers. Design so that is boring.

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Restart during user activity | Dropped sessions, in-flight jobs | Health checks + graceful shutdown; sticky sessions only if needed; prefer short RPO jobs on worker |
| Migration on every auto-update | Lock/contention or failed half-apply | Migrations forward-only and fast; `up.sh` runs migrate before app ready; fail closed if migrate fails |
| DB/sidecar yanked by watcher | Data plane outage | **Never** include DB/proxy in `autoupdate.sh` watch list |
| Bad `latest` published | Home hosts pull a broken candidate | Smoke-test before enabling watcher; pin host to known-good tag/digest and rerun `up.sh`; keep immutable channel clean |
| Overlapping update cycles | Double migrate / flapping | `flock`; one timer per stack |
| K8s pulling mutable tags | Invisible drift | Forbidden on required channel — digests only |
| Refresh rewrites sha tags | Breaks audit “what ran” | Refresh only `latest` / `refresh-*` |
| Major dependency merge | Behavior change under automerge | Majors never auto-merge; AI/human review prompt (see below) |
| Probe failures during rollout | Thrash | Readiness separate from liveness; give migrate+boot headroom |

### Immutable channel: impact control

- Promote only by merging release-pin PRs.
- Roll out with `kubectl rollout status` (or Compose health wait).
- Keep previous pin commit easy to revert.
- Separate secrets/runtime env from pin files so rollback does not scramble config.

### Mutable channel: impact control

- Optional; default off until smoke-tested.
- Watch only app services.
- Always restart through `up.sh`.
- Log digests before/after for deliberate rollback.
- Disable timer independently per stack.

## Major dependency updates (manual lane)

Patch/minor may auto-merge. **Majors stay manual.** Suggested AI review prompt:

```text
Review Dependabot PR #<PR_NUMBER> in <owner>/<repo>.

Goal:
- determine whether the major upgrade is safe to merge
- make any code, workflow, or test changes required for compatibility
- run the repo's existing validation commands
- recommend: approve, needs follow-up, or close/reject

Required work:
1. Read the PR diff; identify dependency and version delta.
2. Check upstream breaking changes relevant to this repo.
3. Fix impacted usage if needed.
4. Run existing validation; do not invent new test tools.
5. Summarize changes, validation, risks, and merge recommendation.

Constraints:
- do not weaken validation to pass
- prefer precise fixes over broad refactors
```

## CVE coverage boundaries

| Covered by refresh rebuild | Covered by pin/dep automation | Not covered without new automation |
| --- | --- | --- |
| Packages inside the same pinned base digest | Lockfile deps, Actions, Dockerfile/Compose/K8s digest bumps | Ecosystems not in Dependabot |
| Dockerfile `apt`/`apk`/downloads against current pins | Lane B newer same-tag base digests | Manual operator bundles (e.g. some K8s operators) |

## Definition of done for LCM on an app

- [ ] Immutable Compose pins file committed
- [ ] Raw Kubernetes variants committed and included in release-pin updates
- [ ] `v*` publish opens release-pin PR
- [ ] Weekly refresh does not mutate sha tags
- [ ] Rollback documented
- [ ] If home auto-refresh enabled: `autoupdate.sh` + `up.sh` + timer docs + DB excluded
- [ ] App `docs/LCM.md` links to this standard and states registry profile
