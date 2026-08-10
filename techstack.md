# Technology stack decision guide

Decision matrix for application shape and core tooling. Operational CI, scanning, lifecycle, and GitHub settings live in sibling docs:

- [`ci.md`](./ci.md) — builds, publish, registries, themes at build time
- [`lcm.md`](./lcm.md) — refresh, deploy channels, workload impact
- [`scanning.md`](./scanning.md) — Dependabot, Trivy, Xray, exceptions
- [`repo-setup.md`](./repo-setup.md) — repository settings and layout checklist
- [`templates/compose-autoupdate/`](./templates/compose-autoupdate/) — canonical vendored Compose mutable-channel template

## Select the application profile

| If this is true | Then use this profile | Required shape |
| --- | --- | --- |
| One Node process, no browser product UI, no database, no second runtime | Tiny standalone app | Node ESM + TypeScript where warranted; one package; no workspace; Dockerfile only when deployed |
| Browser UI only: no owned product API, no persistent shared data domain, no second deployable runtime | Browser app | Vite + TypeScript; one package unless shared runtime code appears; plain DOM/CSS by default; **React 19** when UI state, forms, routing, or non-trivial remote data justify it; optional narrow plain-Node sidecar for ops only; Nginx (or equivalent) static deploy when deployed |
| Owns an API, relational data, shared contracts, a worker, or more than one deployable runtime | Product app | pnpm + Turbo workspace with `apps/web`, `apps/api`, optional `apps/worker`, and `packages/schemas` or `packages/contracts` |

### Browser app UI choice (within the profile)

| If this is true | Then use | Do not |
| --- | --- | --- |
| Mostly static or generated artifacts, light DOM, little client state | Vite + TypeScript + plain DOM/CSS | Add React “just in case” |
| Application state, forms, multi-route UI, or non-trivial server data | React 19 + Vite + TypeScript | Stay on ad-hoc DOM once component complexity is real |
| Styling for either path | `@mkronvold/themes` when the UI is a maintained product surface | Hand-forked theme copies |

## Ask only these questions

| If the request does not answer this | Then ask | If yes | If no |
| --- | --- | --- | --- |
| Durable shared data | Does it persist relational product data or change shared state? | Product app + Postgres | Browser app (or tiny standalone) |
| Server boundary | Does it need an API this app owns? | `apps/api` + Fastify (default) | Browser app only |
| Browser UI complexity | Does the browser surface need app state, forms, routing, or non-trivial remote data? | Browser app + React 19 | Browser app + plain DOM/CSS |
| Background work | Does it need jobs outside an HTTP request lifecycle? | `apps/worker` | No worker |
| Trust boundary inputs | External API data, files, or client requests? | Zod boundary schemas | Internal TypeScript types may suffice |
| Shared types across runtimes | Do web/API/worker share schemas? | `packages/schemas` or `packages/contracts` | Keep types local |
| Deploy audience | Home lab only, WTG/org, or both? | See registry profile in [`ci.md`](./ci.md) | Still pick one primary profile |

## Select the repository layout

| If this is true | Then use | Do not |
| --- | --- | --- |
| One deployable runtime, no shared package | One `pnpm` package | A workspace created only for appearance |
| Web and API both present | `apps/web` + `apps/api` + `packages/schemas` | Frontend imports from the API source tree |
| Independent schedule/retries/deploy lifecycle for jobs | Add `apps/worker` | Long jobs inside web request handlers |
| Shared config across apps | `packages/config` | Duplicated env parsing |
| Containerized product app | `infra/docker/` (or equivalent) for Dockerfiles, Compose prod/dev, helper scripts | Ad-hoc Dockerfile paths without a documented home |
| Cluster deploy (required for product apps in this baseline) | `infra/k8s/` raw manifests/kustomize variants updated by release-pin PRs | Mutable tags in cluster manifests |

## Select the core technologies

| If this is true | Then use | Default or constraint |
| --- | --- | --- |
| New or restructured multi-package app | Corepack-managed `pnpm@10.x` + Turbo | Commit lockfile; `pnpm install --frozen-lockfile` in CI and Docker |
| Existing stable standalone npm app | Keep npm until a justified restructure | `npm ci`; declare Node and npm versions |
| Browser app with app state/forms/routing/server data | React 19 + Vite + TypeScript | React Router for multi-route; TanStack Query for non-trivial remote state |
| Browser app that is mostly viz/static artifacts | Vite + TypeScript without React | Add React only when component/state complexity justifies it |
| New API with ordinary HTTP modules | Fastify + TypeScript | Default backend |
| Strong DI/modules/larger-team boundaries | NestJS on Fastify | Do not use Nest only for uniformity on a small API |
| Existing Express API | Keep while extracting contracts/tests/boundaries | Migrate to Fastify only deliberately |
| Relational domain / migrations / multi-runtime DB access | Postgres + Prisma | Default persistent-data choice |
| Query-first narrow SQL surface | Postgres + `pg` | Explicit SQL; typed at boundary |
| Request/response/file/external input | Zod | Validate at boundary before domain logic |
| Schema used by more than one runtime | Shared Zod in `packages/schemas` or `packages/contracts` | Infer TS types from schemas; do not duplicate models |
| Browser product styling | `@mkronvold/themes` package | Pin version in lockfile; Dependabot updates; regenerate/check committed CSS only if the app vendors a built artifact |

## Themes (`mkronvold/themes`)

| Rule | Detail |
| --- | --- |
| Source of truth | Published package `@mkronvold/themes` (versioned), not a floating git checkout of `main` |
| App consumption | Add as a normal pnpm/npm dependency; import CSS variables or tokens from the package |
| CI | Install from lockfile with frozen install; do not network-fetch themes outside the package manager |
| Stale CSS | If the package ships generated `theme.css`, apps must not hand-edit a fork; bump the dependency instead |
| Local packaging until public publish exists | Use workspace protocol, GitHub Packages, or a private registry — document the feed in app `docs/` and [`ci.md`](./ci.md) |

Until `@mkronvold/themes` is published non-private, treat that as a baseline blocker for new apps (see [`remediation.md`](./remediation.md)).

## Required commands and checks

| If this is true | Expose | CI requirement |
| --- | --- | --- |
| Maintained browser, API, worker, or shared package | `dev`, `build`, `lint`, `typecheck`, `test`, `format` | CI invokes every applicable command |
| Tiny standalone app | `start`, `check`, and `build` when an artifact exists | CI runs `check` and deployable build |
| New TypeScript tests | Vitest | Testing Library for React UI; API/domain tests for server behavior |
| Existing Jest | Keep until migration has clear value | Do not add a second runner without a plan |
| Formatting | Prettier via repo `format` | `format:check` in CI |
| Linting | ESLint | Existing Oxlint acceptable while CI coverage stays equivalent |

## Container and runtime decisions

| If this is true | Then use | Required constraint |
| --- | --- | --- |
| Node API or worker | Official Node **26** `bookworm-slim` build/runtime | Pin `tag@sha256:digest`; multi-stage; runtime-only packages in final image |
| Static web output | Official Nginx runtime image | Pin `tag@sha256:digest`; copy only built assets |
| Uncertain native/OS compatibility | Debian/Ubuntu slim family | Prefer compatibility over Alpine-only size goals |
| Alpine/distroless proposed | Only after build, test, scan, and ops work without shims | Do not choose solely for size |
| Local multi-service dev | Docker Compose | Health checks and volumes; keep prod Compose app-specific |
| Multiple deployed services | One container per service | Separate web, API, worker, DB, migrate/init |

## Registry and deployment profile matrix

Use one profile per runtime path. The required immutable release-pin path is
always present; mutable Compose auto-update is a separately enabled,
single-operator option.

| Profile | Publish source and gate | Runtime pull host | Mutable channel | Immutable release pins |
| --- | --- | --- | --- | --- |
| `ghcr-dev` | Protected GitHub publish with Trivy High/Critical gate | `ghcr.io` using host Docker credential store | Exact `ghcr.io/<namespace>/<image>:latest`, allowlisted app services only | `ghcr.io/...:tag@sha256:digest` through reviewed release-pin PR |
| `artifactory-repo-ops` | Protected `sv4.art/repo.ops` publication after Trivy and Xray; record source-to-pull digest mapping | **`repo.ops` only**, with pull-only host credentials | Exact mutable `repo.ops/...:tag`, `linux/amd64`, mapped source digest, and matching OCI revision | `repo.ops/...:tag@sha256:digest` through reviewed release-pin PR |

`sv4.art` is a protected publication/provenance endpoint, never a host runtime
pull endpoint. The canonical auto-update template rejects it in runtime image
references. Neither profile permits an auto-updater to touch a database,
proxy, migration job, cache, or sidecar.

## Version alignment

| Surface | Rule |
| --- | --- |
| Node major | One supported major in `engines`, `.node-version` (or equivalent), CI, Dockerfiles, sidecars |
| Forward target | Node **26** |
| Temporary mismatch | Document in `security/exceptions.md` with owner and removal date |
| Lockfile | Committed; frozen installs in CI/Docker |
| Base images | `tag@sha256:digest` in Dockerfiles, Compose third-party services, and Kubernetes manifests |

## Minimal decision sequence

| Step | If true | Then |
| --- | --- | --- |
| 1 | No API, no persistent data, no shared state, but has a browser UI | Browser app (plain DOM default; React if UI complexity warrants) |
| 2 | API, relational data, shared contracts, or independent jobs | Product app |
| 3 | Product app | pnpm + Turbo + React/Vite web + Fastify API + Zod schemas + Postgres/Prisma + `@mkronvold/themes` |
| 4 | Explicitly SQL-centric narrow data | Replace Prisma with `pg`; keep Postgres + Zod |
| 5 | Complex module/DI API | NestJS on Fastify |
| 6 | Choice unstated | Use defaults here; ask only the matching question above |

## Out of scope here

Security lanes, scanners, automerge, registries, release pins, and host auto-update are specified in [`scanning.md`](./scanning.md), [`ci.md`](./ci.md), and [`lcm.md`](./lcm.md).
