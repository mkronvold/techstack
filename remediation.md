# Remediation: per-app alignment to the techstack baseline

Bring every maintained application **inline with** [`techstack.md`](./techstack.md), [`ci.md`](./ci.md), [`lcm.md`](./lcm.md), [`scanning.md`](./scanning.md), and [`repo-setup.md`](./repo-setup.md).

This file is **not** split by home vs WTG. Each app has one target profile, one registry profile, and a concrete work list. Use it as the kickoff brief for a remediation session.

## How to run a session

1. Open the app repo (or create a session against it).
2. Paste the **Session kickoff** block for that app.
3. Implement against the standard docs linked above; do not re-invent policy.
4. When done, update this file (status lines, work items, and the snapshot/priority tables).

## Inventory

| App | Repo(s) | Target profile | Registry profile | Intentional exceptions |
| --- | --- | --- | --- | --- |
| themes | `mkronvold/themes` | Shared package (not an app profile) | Publish feed apps can install | N/A — must become a real package |
| content-viewer | `mkronvold-wtg/content-viewer` | Tiny standalone | WTG (Artifactory primary when containerized) | No React/Vite/workspaces |
| kpeviz | `mkronvold-wtg/kpeviz` | Browser app (plain DOM) | WTG | No React/product API unless scope grows |
| revu | `mkronvold/revu` | Product app | Home (GHCR) | `pg` instead of Prisma; npm until migration |
| tavi | `mkronvold-wtg/tavi` (and any home fork) | Product app | WTG (Artifactory primary, GHCR optional) | NestJS on Fastify; Jest on API until Vitest migration |
| whiplash | `mkronvold-wtg/whiplash` | Product app | WTG | Express until after workspace/contracts/tests |

---

## Universal baseline (every containerized or maintained app)

Apply unless the app section explicitly waives an item.

| Area | Required |
| --- | --- |
| Standard pointer | App `docs/` links to `https://github.com/mkronvold/techstack` and lists only **deltas** |
| Node | One major everywhere (forward target **26**): `engines`, CI, Dockerfiles, sidecars |
| Commands | Expose and **CI-run** applicable `build`, `dev`, `lint`, `test`, `typecheck`, `format` |
| Lockfile | Committed; frozen install in CI/Docker |
| Dependabot | Weekly npm/pnpm + Actions + Docker; non-major groups; majors manual |
| Images | Multi-stage; `tag@sha256` pins; SBOM/provenance on publish |
| Trivy | Per-service SARIF; **fail High/Critical**; `.trivyignore.yaml` **and** `security/exceptions.md` |
| Lane B | Base digest-fix bot + crit/high automerge when green |
| Refresh | Weekly image refresh without mutating sha tags |
| Immutable channel | Compose pins + raw Kubernetes + `v*` release-pin PR |
| Home autoupdate | Optional only; never replaces immutable pins |
| Themes (UI apps) | Depend on published `@mkronvold/themes` via lockfile |
| WTG registry | Protected `sv4.art/repo.ops` publication, `repo.ops` runtime pulls only, Xray gate + `.xrayignore` + human exceptions |
| Home registry | GHCR primary; declare profile in docs |

**Universal session add-on (append to any kickoff):**

```text
Also enforce the mkronvold/techstack universal baseline: Node 26 alignment,
frozen installs, Dependabot groups, digest-pinned bases, Trivy fail High/Critical
with both .trivyignore.yaml and security/exceptions.md, Lane B digest automation,
weekly refresh, immutable Compose+Kubernetes release pins on v* tags, and the
correct registry profile (home=GHCR, WTG=Artifactory primary + Xray). Link app
docs to https://github.com/mkronvold/techstack and document only deltas.
```

---

## `mkronvold/themes`

| Field | Value |
| --- | --- |
| Status | **P0 blocker** for UI apps |
| Target | Versioned `@mkronvold/themes` package apps install via pnpm/npm |
| Not | Floating `git clone` / curl of `main` at app build time |

### Work

1. Remove or override `"private": true` for the chosen publish channel (public npm or GitHub Packages).
2. Add versioned release workflow (tag → publish).
3. Export stable paths for `theme.css` / tokens; document breaking-change policy.
4. Point README at `mkronvold/techstack`; drop “adopt stack guide in themes”.
5. After publish, bump consumers (revu, tavi, whiplash, kpeviz) off vendored CSS.

### Session kickoff

```text
Align mkronvold/themes to https://github.com/mkronvold/techstack.

Goal: publish a versioned @mkronvold/themes package apps can pin in lockfiles.

Required:
1. Choose publish channel (GitHub Packages or public npm) and implement release CI on version tags.
2. Ensure package exports the generated theme.css / tokens consumers need.
3. Keep theme.json as source of truth; theme.css generated and checked.
4. Update README to point at mkronvold/techstack; document install + upgrade.
5. Do not change visual tokens unless required for packaging.

Out of scope: rewriting every consumer in this session (list follow-ups only).
```

---

## `mkronvold-wtg/content-viewer`

| Field | Value |
| --- | --- |
| Profile | **Tiny standalone app** (approved) |
| Keep | Node ESM, `server.mjs` / `extension.mjs`, low ceremony, Copilot canvas compatibility |
| Do not | React, Vite workspace, monorepo “for uniformity” |

### Work

1. Document the tiny-standalone exception in README/`docs` with link to techstack profiles.
2. Align Node major declaration; digest-pin deployment image if Dockerized.
3. Minimal CI: `check` (e.g. `node --check`), image build/scan if deployed.
4. Dependabot for npm (if any) + Actions + Docker as applicable.
5. If org-deployed: WTG registry + Xray; if not containerized, document “no image channel”.
6. Skip product-app K8s/release-pin **only** if there is no deployable container — otherwise apply scaled-down immutable tags.

### Session kickoff

```text
Align mkronvold-wtg/content-viewer as the approved tiny standalone profile per
https://github.com/mkronvold/techstack/blob/main/techstack.md.

Keep Node ESM, server/extension entrypoints, and node --check style validation.
Do not add React, Vite, TypeScript monorepos, or workspaces.

Add only: declared Node version, lockfile/deterministic install if packages exist,
CI check + container build/scan if deployed, Dependabot where applicable,
digest-pinned base image, docs pointing at techstack with the tiny-app exception
explicitly justified.
```

---

## `mkronvold-wtg/kpeviz`

| Field | Value |
| --- | --- |
| Profile | **Browser app** (plain DOM/CSS — not React) |
| Keep | Vite + TS viz, Nginx static, narrow Node health/KIMS sidecar, file-backed reports |
| Do not | React, full product API framework, monorepo unless sidecar lifecycle truly splits |

### Work

1. Document browser-app + plain-DOM choice; Node **26** everywhere (today often 24).
2. Named scripts: `typecheck`, `lint`, `test` (even thin), `build`, `format` as applicable; CI runs them.
3. Zod (or equivalent) validation for KIMS responses, report JSON, request queue files.
4. Replace vendored themes with `@mkronvold/themes` after themes publish.
5. Digest-pin bases; Dependabot; Trivy fail-closed + both exception files.
6. Lane B + weekly refresh; WTG Artifactory + Xray for published images.
7. Immutable Compose pins + K8s (or document single-service k8s) + release-pin on `v*`.
8. Optional home autoupdate only if a home mutable channel exists.

### Session kickoff

```text
Align mkronvold-wtg/kpeviz to https://github.com/mkronvold/techstack as a browser
app (plain DOM/CSS, not React).

Keep: Vite+TS visualization, Nginx static deploy, Node health API + KIMS worker
sidecars, file-backed reports/queue.

Required:
1. Node 26 alignment across engines, CI, Dockerfiles.
2. Explicit typecheck/lint/test/build scripts; CI invokes them.
3. Schema validation for KIMS/report/request artifacts.
4. Digest-pinned images, Dependabot, Trivy High/Critical fail with
   .trivyignore.yaml + security/exceptions.md.
5. WTG registry profile (Artifactory primary + Xray) when publishing images.
6. Weekly refresh, Lane B digest automation, immutable release pins for Compose
   and Kubernetes.
7. Plan @mkronvold/themes package dependency once themes is published; stop
   deepening vendored theme forks.

Do not introduce React or a full product API framework unless scope explicitly expands.
```

---

## `mkronvold/revu`

| Field | Value |
| --- | --- |
| Profile | **Product app** |
| Registry | **Home / GHCR** (declare explicitly) |
| Keep | Fastify, React 19, Vite, Zod contracts, direct `pg`, strong Lane B + `autoupdate.sh` |
| Migrate later | npm → pnpm+Turbo (plan or dated exception) |

### Work

| ID | Action |
| --- | --- |
| RV-2/3 | Add Compose image pin file + raw `infra/k8s` variants; `v*` release-pin PR |
| RV-4 | Trivy fail High/Critical; `.trivyignore.yaml` + full `security/exceptions.md` |
| RV-9 | Bot token required so Lane B PRs run CI |
| RV-1 | pnpm+Turbo migration plan **or** dated legacy-npm exception in exceptions/docs |
| RV-5 | `@mkronvold/themes` lockfile dep after TH-1 |
| RV-6–8 | Docs: home profile + link to techstack; deltas only |

### Session kickoff

```text
Align mkronvold/revu to https://github.com/mkronvold/techstack as a product app
on the home/GHCR registry profile.

Keep: Fastify API, React 19 + Vite web, Zod contracts, direct pg, existing Lane B
and autoupdate.sh polish.

Priority order:
1. Immutable channel: compose-prod image pins + raw Kubernetes manifests + v*
   release-pin PR (tavi pattern).
2. Scanning: fail Trivy High/Critical; maintain .trivyignore.yaml AND
   security/exceptions.md with owner/expiry.
3. Require bot token for Lane B PRs that must trigger checks.
4. Document home/GHCR profile and link docs to techstack (deltas only).
5. Either produce a low-risk pnpm+Turbo migration plan without implementing, or
   record a dated npm exception.
6. After themes publish: depend on @mkronvold/themes via lockfile.

Do not weaken Lane B or host autoupdate behavior.
```

---

## `mkronvold-wtg/tavi` (product reference)

| Field | Value |
| --- | --- |
| Profile | **Product app** (reference layout) |
| Registry | **WTG / Artifactory primary**, GHCR optional mirror |
| Keep | pnpm+Turbo, apps/web|api|worker, schemas, release-pin, K8s, fail-closed Trivy, NestJS |
| Add | Lane B, Artifactory+Xray, human exceptions file, themes package |

### Work

| ID | Action |
| --- | --- |
| TV-2/3 | Dual or primary Artifactory publish; Xray gate; pins use primary registry |
| TV-1 | Port revu Lane B digest-fix + crit/high automerge; bot token |
| TV-4 | Add `security/exceptions.md` alongside `.trivyignore.yaml` |
| TV-5 | `@mkronvold/themes` dependency |
| TV-8/9 | Publish path filters include security/ops pins; thicken GITHUB-SETUP |
| TV-6/7 | Link docs to techstack; autoupdate remains optional with smoke-test gate |
| — | Ensure CI **runs tests** (not only lint/typecheck) on every PR |

### Session kickoff

```text
Align mkronvold-wtg/tavi to https://github.com/mkronvold/techstack as the product
app reference on the WTG registry profile.

Keep: pnpm+Turbo workspace, NestJS/Fastify API, worker, Zod schemas, release-pin
PR, Kubernetes digests, fail-closed Trivy.

Priority order:
1. Protected Artifactory `sv4.art/repo.ops` publication and `repo.ops` runtime
   distribution (+ optional GHCR mirror); release pins reference `repo.ops`.
2. Xray policy gate on published digest; .xrayignore + security/exceptions.md.
3. Lane B base-image CVE digest PR bot + crit/high automerge (from revu), with
   bot token so checks run.
4. Normalize exceptions: .trivyignore.yaml AND security/exceptions.md.
5. CI runs full test suite; GITHUB-SETUP documents bot token, auto-merge, delete
   branches, required checks.
6. Depend on @mkronvold/themes once published.
7. Link docs to techstack; keep autoupdate.sh optional and documented.

Do not remove immutable release-pin or K8s pin discipline.
```

---

## `mkronvold-wtg/whiplash`

| Field | Value |
| --- | --- |
| Profile | **Product app** (currently single-package — must grow into workspace) |
| Registry | **WTG** |
| Keep short-term | Express 5, Prisma, deployment split web/API/Postgres/init |
| Do not yet | Fastify migration before boundaries/contracts/tests/Node alignment |

### Work

1. **Node 26** single major (eliminate 22/24 mix across CI vs Docker).
2. Workspace extraction plan → implement: `apps/web`, `apps/api`, `packages/schemas` (or contracts); pnpm+Turbo preferred for new structure.
3. Zod contracts for planner, monthly records, settings-transfer/import-export, catalog, reports, KIMS payloads.
4. Targeted API/domain tests (planner, settings transfer, KIMS errors); CI runs them.
5. Replace vendored themes with `@mkronvold/themes`.
6. Full CI/LCM/scanning baseline: Dependabot groups, digest pins, Trivy fail-closed + both exception files, Lane B, weekly refresh.
7. WTG Artifactory + Xray; immutable Compose pins + K8s + release-pin PR.
8. `docs/CI.md`, `docs/LCM.md`, `docs/GITHUB-SETUP.md` with techstack links.
9. Revisit Express → Fastify **only after** 2–4 are stable.

### Session kickoff

```text
Align mkronvold-wtg/whiplash to https://github.com/mkronvold/techstack as a
product app (WTG registry profile).

Current state: single package, React 19 + Vite web, Express 5 + Prisma API,
split deploy containers, Node major drift, vendored themes, weak formal contracts/tests.

Priority order:
1. Align Node 26 across engines, CI, Dockerfiles.
2. Extract workspace boundaries: apps/web, apps/api, packages/schemas|contracts
   (prefer pnpm+Turbo). Preserve deploy split (web, API, Postgres, init).
3. Add Zod (or equivalent) request/response schemas for planner, settings
   transfer, KIMS, import/export, reports.
4. Add targeted API/domain tests; wire lint/typecheck/test/build into CI.
5. Digest-pinned bases, Dependabot, Trivy High/Critical fail with both exception
   files, Lane B, weekly refresh.
6. Artifactory primary + Xray; immutable Compose + Kubernetes release pins on v*.
7. Plan @mkronvold/themes dependency after themes publish.
8. Keep Express initially; Fastify only as a later follow-up after boundaries,
   contracts, tests, and Node alignment are done.

Do not big-bang rewrite UI or domain behavior.
```

---

## Suggested multi-session order

1. **themes** — unblocks UI consumers  
2. **tavi** — finish WTG product reference (Artifactory, Xray, Lane B)  
3. **revu** — finish home product reference (immutable channel, scan gate)  
4. **whiplash** — largest structural gap (workspace + contracts + baseline ops)  
5. **kpeviz** — browser-app hardening without React  
6. **content-viewer** — document tiny profile + minimal ops  

## Snapshot (high level)

| App | Stack shape toward standard | Ops/LCM toward standard |
| --- | --- | --- |
| themes | packaging gap | publish CI gap |
| content-viewer | intentionally tiny | minimal CI/scan if deployed |
| kpeviz | browser/DOM OK; Node/scripts/validation gaps | needs full container LCM/scan/WTG |
| revu | product OK; npm legacy | strong Lane B; weak immutable+K8s+fail-closed |
| tavi | product reference | strong immutable; weak Lane B+Artifactory+Xray |
| whiplash | farthest on structure | needs almost full baseline |

## Priority ID quick reference

Use these IDs in commits/PRs when useful; details and kickoffs are in the app sections above.

| ID | App | Pri | Gap |
| --- | --- | --- | --- |
| TH-1 | themes | P0 | Publish versioned `@mkronvold/themes` (not private-only) |
| TH-2 | themes | P0 | Release/publish CI on version tags |
| TH-3 | themes | P1 | Consumers stop vendoring CSS; use package |
| TH-4 | themes | P2 | README points at techstack |
| CV-1 | content-viewer | P2 | Document tiny-standalone exception + minimal CI/scan |
| KP-1 | kpeviz | P1 | Node 26, scripts, schema validation, full WTG ops baseline |
| RV-1 | revu | P1 | pnpm+Turbo plan or dated npm exception |
| RV-2 | revu | P0 | Immutable release-pin on `v*` (Compose) |
| RV-3 | revu | P0 | Raw Kubernetes manifests in pin updates |
| RV-4 | revu | P1 | Trivy fail High/Critical + both exception files |
| RV-5 | revu | P1 | `@mkronvold/themes` lockfile dep |
| RV-9 | revu | P1 | Bot token required for Lane B CI |
| TV-1 | tavi | P0 | Lane B digest bot + crit/high automerge |
| TV-2 | tavi | P0 | Artifactory primary (+ optional GHCR) |
| TV-3 | tavi | P0 | Xray gate + `.xrayignore` + human exceptions |
| TV-4 | tavi | P1 | `security/exceptions.md` with `.trivyignore.yaml` |
| TV-5 | tavi | P1 | `@mkronvold/themes` dep |
| WP-1 | whiplash | P0 | Workspace + contracts + tests + Node 26 + full ops baseline |

## Promptlib

Do **not** keep a second full copy under `~/.promptlib`. Entry point:

- `~/.promptlib/techstack.md` — links to this file and sibling policy docs only
