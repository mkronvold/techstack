# Remediation backlog (existing apps)

Gaps versus this baseline. Use for dedicated remediation sessions (e.g. a **revu** session, a **tavi** session, a **themes** publish session). Do not treat this file as runtime policy — standards live in the sibling docs.

## Legend

| Priority | Meaning |
| --- | --- |
| P0 | Blocks claiming compliance with this baseline |
| P1 | Required for low/no-touch parity |
| P2 | Align docs/naming/quality |

---

## `mkronvold/themes`

| ID | Pri | Gap | Target |
| --- | --- | --- | --- |
| TH-1 | P0 | `package.json` has `"private": true` — not a consumable versioned dependency | Publish `@mkronvold/themes` (GitHub Packages or public npm); remove private or publish despite private via GH Packages |
| TH-2 | P0 | No CI publish pipeline for version tags | Add release workflow; signed/immutable versions apps can pin |
| TH-3 | P1 | Apps still vendor or ad-hoc refresh CSS | Document consumption via package export paths; changelog for token breaks |
| TH-4 | P2 | Older promptlib notes said “adopt stack guide in themes” | Point themes README at `mkronvold/techstack` instead |

---

## `mkronvold/revu`

| ID | Pri | Gap | Target |
| --- | --- | --- | --- |
| RV-1 | P1 | npm workspaces, not pnpm+Turbo | Either document as **legacy npm exception** with removal date, or migrate to pnpm+Turbo per [`techstack.md`](./techstack.md) |
| RV-2 | P0 | No immutable release-pin PR on `v*` for Compose + Kubernetes | Add pin file + raw k8s variants + release-pin job (tavi pattern) |
| RV-3 | P0 | No raw Kubernetes manifests in baseline layout | Add `infra/k8s` (or equivalent) updated by release pins |
| RV-4 | P1 | Trivy report-only; no fail on High/Critical | Move to fail-closed + `.trivyignore.yaml` + expand `security/exceptions.md` |
| RV-5 | P1 | Themes not a lockfile dependency | Depend on published `@mkronvold/themes` once TH-1 done |
| RV-6 | P2 | GHCR-only (OK if home profile) | Explicitly declare **home/GHCR** profile in docs; add Artifactory only if WTG deploy appears |
| RV-7 | P2 | Strong Lane B + autoupdate — **keep** as reference | Ensure docs link to `mkronvold/techstack` and drop duplicated policy drift |
| RV-8 | P2 | Local docs (`CI`, `LCM`, `GITHUB-SETUP`, `DOCKER`) are good | Add “Standard: mkronvold/techstack” section; list only deltas |
| RV-9 | P1 | Bot token documented as optional | Treat bot token as required for Lane B CI-on-PR reliability |

**Revu session focus (suggested):** RV-2, RV-3, RV-4 first (immutable channel + scan gate), then RV-1/RV-5.

---

## `mkronvold-wtg/tavi`

| ID | Pri | Gap | Target |
| --- | --- | --- | --- |
| TV-1 | P0 | No Lane B `base-image-cve-fix` / automerge-crit-high | Port revu Lane B workflows; wire bot token |
| TV-2 | P0 | Artifactory primary not implemented (GHCR only) | WTG profile: push `sv4.art/repo.ops`, optional GHCR mirror; pins use primary registry |
| TV-3 | P0 | Xray gate missing | Policy on Artifactory digest; `.xrayignore` + `security/exceptions.md` rows |
| TV-4 | P1 | Exceptions are `.trivyignore.yaml` only | Add/normalize `security/exceptions.md` human registry (keep trivyignore) |
| TV-5 | P1 | Themes not lockfile dependency | Depend on `@mkronvold/themes` after TH-1 |
| TV-6 | P2 | Release-pin + K8s + fail-closed Trivy — **keep** | Link docs to techstack; align naming with standard |
| TV-7 | P2 | `autoupdate.sh` exists; ensure docs match optional-home rules | Explicit opt-in smoke-test gate before timers |
| TV-8 | P2 | Path filters on publish may skip some ops-only changes | Review paths include security ignore files and k8s/docker pins |
| TV-9 | P1 | Repo settings doc thinner than revu | Expand GITHUB-SETUP to bot token, auto-approve Actions, delete branches |

**Tavi session focus (suggested):** TV-2 + TV-3 (registry/Xray), TV-1 (Lane B), TV-4/TV-5.

---

## Cross-cutting / promptlib

| ID | Pri | Gap | Target |
| --- | --- | --- | --- |
| PL-1 | P1 | `~/.promptlib/techstack.md` is a monolith + home/wtg forks | Replace with pointer to this repo; keep short local cheat-sheet only |
| PL-2 | P2 | `techstack-home.md` / `techstack-wtg.md` app comparison still valuable | Move curated comparison into this repo or archive with link |
| PL-3 | P2 | `lcm.md` in promptlib is a stub | Delete or redirect to this `lcm.md` |

---

## Suggested session order

1. **themes** — publish package (unblocks RV-5, TV-5).
2. **tavi** — Artifactory + Xray + Lane B (WTG reference completion).
3. **revu** — immutable pins + K8s + Trivy fail-closed (home reference gains required channel).
4. **promptlib** — replace monolith with links to `mkronvold/techstack`.

## Compliance snapshot (at doc authoring)

| Capability | revu | tavi | standard |
| --- | --- | --- | --- |
| pnpm + Turbo product default | no (npm) | yes | pnpm+Turbo |
| Validate lint/typecheck/test/build | yes | yes | required |
| GHCR publish | yes | yes | home primary |
| Artifactory primary | no | no | WTG primary |
| Trivy SARIF | yes | yes | required |
| Fail High/Critical | no | yes | required |
| `.trivyignore` + `security/exceptions.md` | partial | partial | both required |
| Xray | no | no | WTG required |
| Lane B digest bot | yes | no | required |
| Weekly refresh | yes | yes | required |
| Release-pin PR | no | yes | required |
| Kubernetes pins | no | yes | required |
| Home autoupdate.sh | yes | yes | optional |
| Themes package dep | no | no | required when UI |
