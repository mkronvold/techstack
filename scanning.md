# Security scanning and dependency automation

Dependabot, Trivy, Xray, exception registries, and Lane B base-image remediation. CI wiring: [`ci.md`](./ci.md). Deploy impact: [`lcm.md`](./lcm.md).

## Default posture

| Layer | Default |
| --- | --- |
| Dependency updates | Weekly grouped non-major; majors manual; security updates expedited |
| Image scan (all profiles) | **Trivy** on each service image; SARIF to GitHub code scanning; **fail on High/Critical** unless excepted |
| Artifact scan (WTG/Artifactory) | **Xray** policy evaluation on the published primary digest — additional gate |
| Exceptions | **Both** machine ignore file(s) **and** human-readable `security/exceptions.md` |
| Lane B | **Required** for maintained container apps |

## Two event-driven security lanes

Do not model “crit/high CVE” as a third Dependabot group.

| Lane | Signal | Typical trigger | Remediation |
| --- | --- | --- | --- |
| **A. Dependency / Dependabot** | Advisory or newer version for lockfile, Action, or Dockerfile pin | Dependabot alert / security update | Dependabot PR; patch/minor may group + automerge; majors manual |
| **B. Image / scanner** | Finding on built or published image (OS packages, image contents) | Build/publish/refresh scans; Xray on Artifactory digest | Bump base `tag@sha256`, app dep, or Dockerfile content; digest-fix bot PR; rebuild/publish |

Rules:

1. Weekly groups are the routine conveyor for non-major package and Actions updates plus scheduled refresh/scan.
2. Lane A is continuous for **manifest/lockfile** deps. Crit/high with a fix does not wait for Monday.
3. Lane B is continuous for **image** results (code scanning and/or Xray), not a Dependabot group.
4. Publishing a fixed image is not deploy — hosts/clusters follow [`lcm.md`](./lcm.md).
5. Do not defer known crit/high fixes solely because the weekly batch has not run.

## Dependabot shape

```yaml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
    groups:
      npm-non-major:
        applies-to: version-updates
        update-types: [minor, patch]
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
    groups:
      actions-non-major:
        applies-to: version-updates
        update-types: [minor, patch]
  - package-ecosystem: docker
    directory: /apps/api   # one entry per Dockerfile dir / infra path
    schedule:
      interval: weekly
  # Add docker entries for web, worker, Compose-adjacent dirs, and
  # document any Kubernetes image refs Dependabot should own.
```

| Rule | Detail |
| --- | --- |
| Grouping | Non-major npm/pnpm and Actions grouped; majors ungrouped |
| Limit | `open-pull-requests-limit` counts PRs not packages — grouping prevents starvation |
| Automerge | Patch/minor only via workflow; never majors |
| Docker | Weekly pin PRs coexist with Lane B; **security digest PRs supersede** stale Dependabot docker PRs for the same bases |
| Themes | Covered by npm/pnpm entry once `@mkronvold/themes` is a real dependency |

## Trivy (required everywhere images build)

| Requirement | Detail |
| --- | --- |
| When | PR candidate images, default-branch builds, scheduled refresh, and/or published digests |
| Severity gate | **Fail** the scan job on **High** and **Critical** unless excepted |
| SARIF | Upload to GitHub code scanning with **one category per service** (e.g. `trivy-api`, `trivy-web`) |
| JSON artifacts | Retain machine-readable Trivy JSON for Lane B bots and triage |
| Scope | Prefer scanning the image just built / the exact published digest |
| Baseline | New apps may temporarily use report-only **only** with a dated exception while triaging the first baseline; default remains fail-closed |

### Machine exceptions — `.trivyignore.yaml`

- Checked into git at repo root (or documented path).
- Each entry: specific vuln id, statement, owner, `expired_at`.
- No broad permanent suppressions.
- Remove when upstream fix lands.

### Human exceptions — `security/exceptions.md`

Required companion for **every** accepted risk (Trivy and Xray):

| Field | Purpose |
| --- | --- |
| ID | CVE / GHSA / Xray issue id |
| Scanner | `trivy` / `xray` / both |
| Service/image | What it applies to |
| Owner | Human accountable |
| Rationale | Why not fixed now |
| Mitigation | Compensating control |
| Expiry | Date; expired entries block release until renewed or fixed |
| Links | Alert URL, PR, ticket |

**Policy:** a finding is not accepted unless it appears in the machine ignore file **and** is justified in `security/exceptions.md` (or the ignore entry’s statement fully duplicates those fields **and** the human file indexes it). Prefer both files always.

## Xray (required for WTG / Artifactory primary)

| Requirement | Detail |
| --- | --- |
| When | After push of the image digest to `sv4.art/repo.ops` (or org path) |
| Role | Org policy gate on the artifact that clusters pull |
| Failure mode | Fail publish/promote job or block download per org Xray watch/policy |
| Home/GHCR-only apps | Xray not required |
| Dual-push | Gate at least the **primary** Artifactory digest |

### Machine exceptions — `.xrayignore` (or org-standard equivalent)

- Git-tracked list of waived Xray issue ids / components matching org practice.
- Same expiry discipline as Trivy.
- If org Xray UI waivers are mandatory, **mirror** them into git so apps stay reviewable offline.

### Human file

Same `security/exceptions.md` rows with `Scanner: xray` (or both).

### Normalization

Lane B bots may consume Trivy JSON first. When Xray is the only signal for a base package, adapt export/API into the same digest-fix PR flow or document manual pin bumps with the same SLA (crit/high not waiting for weekly batch).

## Lane B — base-image CVE digest automation (required)

Scanner findings do not open Dependabot PRs by themselves. Every maintained container app implements:

| Step | Requirement |
| --- | --- |
| Detect | Scan each deployable image on PR/publish/refresh; SARIF + JSON; Xray on Artifactory primary |
| Classify | Base OS/image vs app dependency in image vs no upstream fix |
| Fix base | Prefer bump `tag@sha256` in Dockerfiles, Compose third-party pins, and K8s third-party pins |
| Automate | Workflow opens/updates **one** digest-only PR when newer same-tag digest reduces fixed CVEs |
| Open policy | Open for any severity when a newer digest helps |
| Merge policy | **Critical/high:** auto-merge when required checks green and diff is digest-only (no major tag jump). **Medium/low:** human merge |
| Dedup | One open bot PR per pin set; push to existing branch |
| Coexist | Security digest PRs supersede stale Dependabot docker PRs |
| Verify | Rebuild/rescan; alerts close or reduce |
| Pause | Optional label/file to pause bot churn for an accepted finding already in the exception registry |

Reference: `mkronvold/revu` `base-image-cve-fix.yml` + `automerge-base-image-cve.yml`.

**Bot token:** PRs opened with `GITHUB_TOKEN` often do not trigger further workflows. Prefer a fine-scoped PAT/app token secret so digest PRs still run validate/publish checks (see [`repo-setup.md`](./repo-setup.md)).

## Cadence diagram

```text
Weekly (batch):
  • npm/pnpm non-major group
  • Actions non-major group
  • docker digest/tag PRs
  • scheduled image refresh + SBOM/scan
  • backlog of medium/low digest PRs and alerts

Event-driven:
  • Lane A: Dependabot/security fix available
  • Lane B: image findings → digest-fix PR (auto-merge crit/high when green)
  • Xray policy fail on Artifactory publish → block/fix before promote
```

## Definition of done for scanning on an app

- [ ] Dependabot.yml with npm/pnpm, Actions, Docker; non-major groups
- [ ] Dependabot alerts + security updates enabled in repo settings
- [ ] Code scanning enabled; Trivy SARIF per service
- [ ] Fail on High/Critical with `.trivyignore.yaml` + `security/exceptions.md`
- [ ] If Artifactory primary: Xray gate + `.xrayignore` (or equivalent) + human file rows
- [ ] Lane B digest-fix + crit/high automerge workflows
- [ ] Automerge workflow for Dependabot patch/minor only
- [ ] Exceptions have owners and expiry dates; no permanent broad ignores
