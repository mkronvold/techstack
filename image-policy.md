# Application image policy

**Organization-wide policy name:** **Upstream-First Application Image Standard
(UFAIS)**.

This standard defines when to deploy an unmodified vendor image and when to
build, publish, and operate a derived application image. It applies to
TypeScript/React (TSX) web applications, Node APIs (including Express), and
their supporting services. It complements the release, registry, scan, and
lifecycle requirements in [`ci.md`](./ci.md), [`scanning.md`](./scanning.md),
and [`lcm.md`](./lcm.md); those documents remain authoritative where they are
more specific.

## Terms and policy statement

| Term | Meaning in this standard |
| --- | --- |
| **Upstream-image-first** | Evaluate an approved, unmodified vendor image before creating a new base or support-service image. It does **not** mean application code must run from an upstream image. |
| **Off-the-shelf / unmodified upstream image deployment** | Deploy an image supplied by its vendor without adding application files, dependencies, or layers. `postgres:16-alpine` is an example when it is digest-pinned. |
| **Derived application image** | An immutable image built from an approved upstream base with this application's code, dependencies, or built artifacts added. This is a custom image even when it is thin. |
| **Thin application image** | A derived image whose runtime stage contains only the runtime, production dependencies, application artifact, and required runtime metadata; it is not a full build environment. |
| **Build container** | A disposable build stage or CI environment that contains compilers, package-manager cache, dev dependencies, source, and build tooling. It is not deployed. |
| **Runtime container** | The final deployed stage. It contains no build cache, source-only tooling, dev dependencies, or build secrets unless a documented runtime need requires them. |

**Default:** use approved, unmodified upstream images for stateful and
supporting services, and use immutable derived application images for
application code.

1. PostgreSQL, Redis, Nginx, Caddy, and similar supporting services use their
   supported vendor image without application layers unless the service itself
   is intentionally customized and reviewed.
2. React/Vite web and Node/Express API workloads build a derived application
   image for each deployable service. A TSX application normally compiles and
   bundles TypeScript, JSX, CSS, and asset imports into static artifacts; its
   runtime should serve those artifacts, not run a development server. A
   Node/Express API normally needs its application code and production
   dependency graph packaged with a compatible Node runtime.
3. An unmodified-image deployment for application code is allowed only through
   the exception process below. “No private registry” does **not** mean “no
   custom images”: an application image built locally from a Dockerfile is a
   derived/custom image whether it is only tagged locally, pushed to GHCR,
   promoted through Artifactory, or never published.

## ReviO/Whiplash reference case

ReviO currently builds local application images from Dockerfiles:
`whiplash-web:latest` and `whiplash-api:latest`. They are **derived application
images** layered on the upstream Docker Hub base
`node:26-bookworm-slim`. PostgreSQL is the separate, unmodified upstream
`postgres:16-alpine` service image. ReviO does **not** pull its application
images from Artifactory or GHCR today.

Therefore, ReviO uses a conventional custom/derived application-image build,
not a zero-image-build model. Its lack of a private registry changes
distribution and retention controls; it does not change the fact that the web
and API images are custom. The target stack shape is also consistent with the
Whiplash product-app profile: a React/Vite web, Express/Prisma API, and split
web/API/PostgreSQL deployment ([`remediation.md`](./remediation.md#L267-L285)).

## Required controls

| Control | Requirement |
| --- | --- |
| Approved source and provenance | Use approved vendor registries and bases. Record the base image, immutable digest, Dockerfile, source revision, build time, and builder/provenance with every published application artifact. Do not substitute an unapproved mirror or an image whose publisher cannot be verified. |
| Digest pinning | Dockerfiles, Compose third-party services, and Kubernetes image references use `tag@sha256:digest` ([`techstack.md`](./techstack.md#L117-L125)). Tags communicate intent; digests select the exact bytes. |
| Base patching | Update approved same-tag base digests through the Docker dependency lane and rebuild/rescan derived images. Critical/high fixes are event-driven; routine pin review is weekly ([`scanning.md`](./scanning.md#L130-L145), [`lcm.md`](./lcm.md#L138-L150)). A rebuild alone does not advance a digest-pinned base. |
| Immutable application artifacts | Build once per source revision, attach source-revision OCI labels, and publish immutable `sha-*` and/or version references. Release deployment references use `tag@sha256:digest`, not mutable tags ([`ci.md`](./ci.md#L138-L147), [`lcm.md`](./lcm.md#L14-L26)). |
| Build and dependency reproducibility | Commit the lockfile and install with `corepack pnpm install --frozen-lockfile` (or `npm ci` for an approved legacy npm application) in CI and build stages ([`ci.md`](./ci.md#L13-L27), [`techstack.md`](./techstack.md#L50-L55)). The declared Node major must agree across engines, CI, and Dockerfiles. |
| Multi-stage and minimal runtime | Separate build and runtime stages. Copy only built web assets or API production dependencies plus code/artifacts into the final image. Do not deploy package-manager caches, compiler toolchains, test files, or source maps unless explicitly needed. |
| Non-root runtime | The final application process must run as a non-root UID. A web server image must listen on an unprivileged port or use a vendor-supported unprivileged image/configuration. Do not grant root merely to make a default port work. |
| Secret handling | Never copy `.env`, credential files, package tokens, SSH keys, or runtime secrets into image layers. Use BuildKit secret mounts for build-only private-feed credentials and inject runtime configuration through the platform's secret/environment mechanism. [`ci.md`](./ci.md#L62-L70) requires the same boundary for private package access. |
| SBOM, scan, signing, provenance | Publish BuildKit SBOM and provenance; retain CycloneDX and/or Trivy JSON for at least 90–180 days ([`ci.md`](./ci.md#L49-L60)). Scan each service image, fail on High/Critical unless a dated exception exists, and use Xray for an Artifactory-primary digest ([`scanning.md`](./scanning.md#L71-L80), [`scanning.md`](./scanning.md#L106-L124)). Sign or attest the published digest with the approved identity, then verify that evidence before promotion/deployment. |
| License verification | Verify base-image, OS-package, and application dependency licenses before approval/promotion. Record incompatible or unknown licenses as release-blocking unless a named legal/security owner grants a dated exception. |
| Runtime configuration | Image contents are environment-neutral. Inject URLs, ports, feature flags, and secrets at runtime; validate required configuration at startup and document non-secret examples. Keep runtime secrets separate from immutable image-pin files ([`lcm.md`](./lcm.md#L126-L134)). |
| Health and readiness | Each application service provides a liveness process check and a readiness endpoint that covers dependencies appropriate for admitting traffic. Compose/Kubernetes must use readiness separately from liveness; startup and migrations receive adequate headroom ([`lcm.md`](./lcm.md#L153-L167)). |
| Cache handling | Scope build cache by image and dependency inputs. Caches are accelerators, never artifact sources of truth; a clean build must succeed. Scheduled security refreshes may use `pull: true` and `no-cache: true` ([`ci.md`](./ci.md#L49-L60)). Never copy a package cache into runtime. |
| Retention and rollback | Retain immutable images, SBOMs, provenance, scan reports, and deployment-pin history for the organization's approved retention period. Roll back by restoring the prior reviewed digest pin; do not retag a mutable image as a substitute ([`lcm.md`](./lcm.md#L14-L27)). |
| Registry distribution | Choose one primary profile: GHCR for home deployments or protected Artifactory publication with `repo.ops` runtime pulls for WTG/org deployments ([`ci.md`](./ci.md#L73-L128)). A local-only image still follows build, scan, and provenance controls; its documented exception identifies how it is retained and recovered. |

## Decision matrix

| Workload | Default image model | Why | Minimum runtime content | Do not |
| --- | --- | --- | --- | --- |
| React/Vite static web | Derived thin application image on an approved static-server runtime | Vite must compile TSX and produce hashed static assets; the runtime serves `/dist` | Static artifact, static-server configuration, non-root server user | Run `vite dev`, install dependencies, or compile TSX at runtime |
| Node/Express API | Derived thin application image on approved Node runtime | API code and production dependency graph are application-specific | Built JS (or reviewed runtime TS strategy), production dependencies, non-root user, startup metadata | Deploy the builder stage, include dev dependencies, or mount source as the production delivery mechanism |
| PostgreSQL | Unmodified upstream vendor image | PostgreSQL runtime is a supporting stateful service; application schema belongs in migrations, not a rebuilt database server image | Vendor image, named/managed volume, runtime configuration and secrets | Auto-update it with application-image tooling or treat a major-version tag change as a harmless rebuild |
| Redis, Nginx/Caddy | Unmodified upstream vendor image by default | Support-service behavior is vendor-owned and separately patchable | Vendor image plus reviewed runtime config/volume mounts where required | Add application code or secrets to image layers |
| Application code under a valid exception | Unmodified upstream image only temporarily | Exception must prove that artifacts/config can be supplied at startup without weakening controls | Vendor image plus read-only mounted artifact/config source and non-root execution | Call the result immutable or reproducible if artifacts are fetched/mutated at startup |

## Exception: deploying application code from an unmodified image

An exception is valid only when **all** of these tests pass:

1. The application artifact is produced by a separate reproducible build,
   versioned and immutable, and supplied through a verified read-only mount,
   platform artifact mechanism, or approved startup volume.
2. Runtime configuration and secrets are injected separately; no bootstrap step
   downloads source, dependencies, or credentials from an uncontrolled network
   location.
3. The upstream image can run the artifact non-root without package installation,
   compilation, or mutation at container start.
4. Deployment records the exact upstream digest **and** artifact digest/version,
   preserves both for rollback, and performs the same SBOM, scan, provenance,
   license, health, and readiness controls.
5. The operational team accepts the additional artifact-distribution, mount,
   permission, and compatibility burden.

The exception record lives with `security/exceptions.md` or an equivalent
reviewed application-local exception register and includes: scope/workload,
rationale, threat and operational limitations, artifact source, owner,
approver, issue link, start date, expiry/review date (maximum 90 days unless
the organization approves otherwise), and tested rollback reference. This
matches the existing owner-and-expiry discipline for image risk exceptions
([`scanning.md`](./scanning.md#L82-L104)).

Graduate immediately to a derived application image when any condition changes:
the application needs a build step, needs Node packages or native modules at
runtime, startup fetches code/dependencies, artifacts cannot be independently
digest-pinned and retained, rollbacks cannot restore the paired image/artifact,
the application needs service-specific hardening, or the exception expires.

An unmodified-image approach can be useful for a short-lived static artifact
mounted into a vendor web server or for a platform that separately manages
immutable artifacts. Its tradeoffs are material: image-only scanners/SBOMs do
not describe the mounted artifact; the artifact-to-image pair adds a deployment
coordinate; startup/mount permissions may fail late; and portability depends on
platform artifact facilities. It is not the default for TSX or Express apps.

## Reference patterns

Replace each `<approved-digest>` with a reviewed digest. These snippets show
the required shape, not deployable pins or real configuration values.

### React/Vite static web: build once, serve static assets

```dockerfile
# syntax=docker/dockerfile:1
FROM node:26-bookworm-slim@sha256:<approved-digest> AS build
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM nginxinc/nginx-unprivileged:stable-alpine@sha256:<approved-digest> AS runtime
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 8080
```

Use a reviewed server configuration when SPA fallback, cache headers, CSP, or
an alternate document root is required. The second stage is a **derived**
application image even though it contains only an upstream Nginx runtime and
the Vite output.

### Node/Express API: package production runtime only

```dockerfile
# syntax=docker/dockerfile:1
FROM node:26-bookworm-slim@sha256:<approved-digest> AS build
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build && pnpm prune --prod

FROM node:26-bookworm-slim@sha256:<approved-digest> AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build --chown=node:node /app/dist ./dist
COPY --from=build --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/package.json ./package.json
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

For a monorepo, copy only the relevant application and production workspace
dependencies, preserving the same frozen lockfile rule. Do not use `latest` in
a Dockerfile or consider a local tag to be a release pin.

### Compose: app images are derived; PostgreSQL stays upstream

```yaml
services:
  web:
    image: registry.example/whiplash-web:v1.2.3@sha256:<web-digest>
    read_only: true
    tmpfs: [/tmp]
    ports: ["8080:8080"]
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:8080/ >/dev/null"]

  api:
    image: registry.example/whiplash-api:v1.2.3@sha256:<api-digest>
    user: "1000:1000"
    read_only: true
    tmpfs: [/tmp]
    environment:
      NODE_ENV: production
      DATABASE_URL: ${DATABASE_URL:?set-at-runtime}
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "node -e \"fetch('http://127.0.0.1:3000/ready').then(r => process.exit(r.ok ? 0 : 1)).catch(() => process.exit(1))\""]

  db:
    image: postgres:16-alpine@sha256:<postgres-digest>
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set-at-runtime}
    volumes: [postgres-data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]

volumes:
  postgres-data:
```

`registry.example` is only a placeholder. For a WTG deployment, use the
approved `repo.ops` runtime name after protected Artifactory publication; for a
home deployment, use the configured GHCR name. Never place credentials in this
file. Keep the image pins in a reviewed release-pin file and supply
`DATABASE_URL`/`POSTGRES_PASSWORD` through the host or deployment secret
mechanism.

## Release checklist

- [ ] Classify every image as upstream/unmodified or derived application image.
- [ ] Pin each base and deployment image by digest; update bases through the
  documented Docker and Lane B process.
- [ ] Use a multi-stage build, frozen lockfile install, minimal non-root runtime,
  and no secrets or cache in final layers.
- [ ] Build, test, scan, create SBOM/provenance, license-check, sign/attest, and
  retain the exact application digest before promotion.
- [ ] Configure liveness/readiness and inject runtime configuration separately.
- [ ] Release through a reviewed immutable pin and retain a tested rollback pin.
- [ ] Record any unmodified-application exception with owner, expiry, review,
  limitations, and graduation trigger.

## External references

- [Docker: build multi-stage images](https://docs.docker.com/build/building/multi-stage/)
- [Docker: image digests and immutable references](https://docs.docker.com/dockerscout/guides/image-digests/)
- [Docker: BuildKit build attestations (SBOM and provenance)](https://docs.docker.com/build/metadata/attestations/)
- [SLSA: provenance](https://slsa.dev/provenance/)
- [OCI image-spec annotations](https://github.com/opencontainers/image-spec/blob/main/annotations.md)
