# RFC-0080: Project-Level Skills Lock File

- **Status**: Draft
- **Author(s)**: Samu Vannier (@samuv)
- **Created**: 2026-07-03
- **Last Updated**: 2026-07-03
- **Target Repository**: toolhive
- **Related Issues**: [toolhive#5715](https://github.com/stacklok/toolhive/pull/5715)

## Summary

Add a project-level skills lock file (`toolhive.lock.yaml`) that pins the name, version, source, resolved reference, and digest of every project-scoped skill install, plus `thv skill sync` and `thv skill upgrade` commands to restore pinned state and pull newer content from the catalog. This brings to skills the reproducibility guarantees that `package-lock.json`, `Cargo.lock`, and `go.sum` provide for other ecosystems.

## Problem Statement

- **Current behavior:** `thv skill install --scope project` writes files to client skill dirs and a SQLite record, but nothing pins *which* content was installed in a shareable, version-controlled form. Two teammates cloning the same repo get whatever the catalog currently serves, not what the original installer intended.
- **Who is affected:** any team using project-scoped skills for shared workflows (code-review conventions, testing skills, org-specific instructions). Reproducibility and controlled upgrades are both missing.
- **Why worth solving:** skills are supply-chain artifacts (instructions an AI assistant follows). Unpinned installs mean silent drift across machines and no auditable path from "what we agreed to use" to "what's actually installed". Every comparable package manager solved this with a committed lock file; skills currently have no equivalent.

## Goals

- Pin the exact content digest of every project-scoped skill install in a file committed to the project repo.
- Restore an identical skill set on any machine via `thv skill sync`.
- Provide a controlled, digest-based upgrade path via `thv skill upgrade`.
- Keep the lock client-agnostic by pinning content, not which client apps installed it.
- Stay out of user-scope installs entirely.

## Non-Goals

- Version-constraint resolution or manifest-driven installs. There is no `toolhive.yaml` with `^1.0.0` ranges in this proposal. The lock file is the declaration of intent; sync installs exactly what's pinned.
- Per-client pinning. The lock pins content; sync installs for all detected clients, overridable with `--clients`.
- A dependency resolver or transitive dependency graph for skill-on-skill dependencies.
- Lock-file entries for user-scope installs.

## Proposed Solution

Use a single lock file written by install commands. `--scope project` installs upsert an entry; uninstalls remove it. `sync` reinstalls at the pinned `resolvedReference@digest`; `upgrade` re-resolves the original `source` and rewrites the entry if the digest changed.

### High-Level Design

```mermaid
flowchart LR
    installCmd["thv skill install --scope project"] --> svc[skillsvc install]
    svc --> files["client skill dirs"]
    svc --> db[(SQLite state)]
    svc --> lock["toolhive.lock.yaml (upsert)"]
    syncCmd["thv skill sync"] --> lock2["read lock"] --> pinned["install resolvedReference@digest"]
    upgradeCmd["thv skill upgrade"] --> reresolve["re-resolve source"] --> compare{"digest changed?"} -->|yes| rewrite["install + rewrite entry"]
```

### Detailed Design

The lock file has a top-level `version: 1` field and a deterministic list of skill entries sorted by name for stable diffs:

```yaml
version: 1
skills:
  - name: code-review
    version: 1.0.0
    source: code-review
    resolvedReference: ghcr.io/org/code-review:1.0.0
    digest: sha256:9f2b1e...
```

Each entry stores `name`, optional `version` from `SKILL.md` frontmatter, `source`, `resolvedReference`, and `digest`. `source` is the original user input, such as a plain registry name, OCI reference, or `git://` reference, preserved verbatim so upgrade can re-resolve it. `resolvedReference` is the concrete OCI reference or git URL that source resolved to. `digest` is either an OCI `sha256:...` digest or a git commit hash.

The schema deliberately omits `installedAt`. Every major lock file, including npm, pnpm, yarn, Cargo, Go, and Poetry, stores identity, source, and integrity, not chronology. Reproducibility is guaranteed by the digest, not a timestamp; timestamps are environment-local and produce meaningless merge churn on no-op regenerations. "When was this pinned" is answered better by `git blame`, and `git log -p -- toolhive.lock.yaml` serves any recency or audit need.

#### Component Changes

- Add a new `pkg/skills/lockfile` package for schema handling, load/save, and file-locked upsert/remove using `pkg/fileutils.WithFileLock` in the same pattern as config writes. Marshalling is deterministic, with sorted entries.
- Update `pkg/skills/skillsvc` install and uninstall hooks so project-scope installs upsert a lock entry recording the original `opts.Name` as `Source` before any internal resolution, and uninstalls remove it. Lock write errors are logged but never fail the install or uninstall because the skill's files and DB record are already correct, and a subsequent sync or upgrade can repair the lock. User-scope installs never touch the lock.
- Add `Sync`, which reads the lock, compares each entry against `SkillStore` state, installs strictly by pinned `resolvedReference@digest` using OCI pull-by-digest or git clone plus checkout of the pinned commit, and reports unmanaged project-scope skills. With `--prune`, it uninstalls unmanaged skills. It never rewrites the lock; it only makes FS/DB match what's pinned.
- Add `Upgrade`, which re-resolves each entry's `source` exactly as a fresh `thv skill install <source>` would. Registry names resolve through the catalog, git branches/tags resolve to current heads, and OCI tags resolve to current digests. If the digest changed, upgrade installs and rewrites the entry. Immutable sources such as OCI `@sha256:` digests or full 40-character git commit hashes are reported as `not-upgradable` without contacting the network. `--dry-run` prints what would change. `source` is never rewritten, so future upgrades keep re-resolving the same input.
- Have `Sync` and `Upgrade` call `installInternal`, not `Install`, so they do not overwrite the entry's `Source` with an already-resolved reference.

#### API Changes

This proposal is additive and introduces no breaking changes:

- `POST /skills/sync` with body `{projectRoot, clients, prune}` returns a report containing installed, up-to-date, unmanaged, pruned, and failed skills.
- `POST /skills/upgrade` with body `{projectRoot, names, dryRun, clients}` returns per-skill outcomes of upgraded, up-to-date, not-upgradable, or failed, including old and new digests where relevant.
- `SkillService` gains `Sync` and `Upgrade` methods, and `pkg/skills/client` gains corresponding HTTP methods.

#### Configuration Changes

None. The lock file is project-level, discovered by auto-detecting the git root from the current working directory, using the nearest enclosing `.git` directory. `--project-root` provides an override. No global config is added.

#### Data Model Changes

No SQLite schema changes are required. The lock file is a new committed YAML artifact at the project root, while the SQLite `installed_skills` table continues to hold runtime install records. The lock is the shareable pin; SQLite is the local install state. `sync` reconciles the two.

## Security Considerations

### Threat Model

A malicious or compromised lock file, such as one introduced through a tampered PR, could pin a skill to a known-bad digest. The threat is an attacker committing a lock entry pointing to malicious skill content that teammates then sync. The relevant attacker capabilities are write access to the project repo or the ability to submit a PR that is merged.

### Authentication and Authorization

Lock-file writes happen server-side in `skillsvc`, gated by the same API auth as existing skill install/uninstall operations. This proposal introduces no new auth surface. `sync` and `upgrade` reuse the existing OCI Docker credential chain and git token auth paths, with no new credential handling.

### Data Security

The lock file contains no secrets, only public references and content digests. It is committed to git by design. No sensitive data is stored or transmitted beyond what `thv skill install` already transmits.

### Input Validation

Lock entries are validated on load via the same `ValidateSkillName` used for installs. `resolvedReference` and `digest` are passed through the existing OCI and git resolution paths, which already enforce SSRF guards such as no localhost/private IPs in git refs except dev mode, shell-injection reference validation, and supply-chain checks such as requiring artifact skill names to match OCI repo paths. A tampered lock can only pin references that would also pass a manual `thv skill install`.

### Secrets Management

None. The lock stores no tokens. Registry and git authentication use the existing per-process credential chain.

### Audit and Logging

Lock upsert/remove errors are logged via `slog.Warn` with skill name and project root. `sync` and `upgrade` return structured reports covering installed, up-to-date, unmanaged, pruned, failed, upgraded, and not-upgradable states suitable for audit. The lock file itself, being git-committed, gives a full history of what was pinned when, which is stronger cross-machine provenance than DB-only audit.

### Mitigations

- Pinned-digest installs mean `sync` reproduces exact bytes, and a merged lock entry is auditable in `git blame`.
- `upgrade` refuses to upgrade entries already pinned to a digest or commit, preventing accidental re-resolution of intentionally locked content.
- Best-effort lock writes mean a failed lock write cannot corrupt an install because files and DB state are already correct, and `sync` can repair drift.

## Alternatives Considered

### Alternative 1: Separate manifest and lock file

- **Description:** A hand-edited `toolhive.yaml` declares desired skills with version constraints, and the lock resolves them.
- **Pros:** Supports `^1.0.0` ranges; upgrade is "re-resolve constraints".
- **Cons:** Requires building a dependency resolver; doubles the surface area with two files to keep in sync; skills do not have a rich enough versioning or constraint ecosystem to justify it yet.
- **Why not chosen:** The POC scope is narrower. The single-lock model delivers reproducibility now and leaves the door open to a manifest layer later without rework.

### Alternative 2: Store pin data only in the SQLite store

- **Description:** No committed file; sync reads from a shared or replicated DB.
- **Pros:** No new file format.
- **Cons:** Not portable across machines; cannot be reviewed in a PR; no `git blame` provenance; defeats the committed, auditable pin goal entirely.
- **Why not chosen:** This approach fundamentally does not meet the reproducibility and audit goals.

### Alternative 3: Per-client lock entries

- **Description:** Pin which client apps each skill installs into.
- **Pros:** Precise per-client control.
- **Cons:** Couples the lock to the local client set; bloats entries; one teammate without Cursor installed cannot sync cleanly.
- **Why not chosen:** Content pinning is the real need; client targeting is a local concern handled at sync time by `--clients` or detected clients.

## Compatibility

### Backward Compatibility

This change is fully backward compatible. Existing project-scope installs simply start writing a lock file. Pre-existing installs without a lock entry still work; sync reports them as unmanaged and does not touch them unless `--prune` is set. No migration is required because the lock file is additive. User-scope installs are entirely unchanged.

### Forward Compatibility

The top-level `version: 1` schema field allows future schema evolution. The `source` field is the extensibility hook for a future manifest/resolver layer; a v2 schema could add constraint expressions alongside `source` without breaking v1 readers. New entry fields can be added with `omitempty`.

## Implementation Plan

A POC implementation already exists in [toolhive#5715](https://github.com/stacklok/toolhive/pull/5715). Post-RFC acceptance, split it into reviewable PRs.

### Phase 1: Lock file package

- Add the `pkg/skills/lockfile` schema and file-locked operations.

### Phase 2: Install/uninstall hooks

- Add project-scope install upsert and uninstall remove behavior.

### Phase 3: Sync

- Add the `Sync` service method, API, and `thv skill sync` CLI.

### Phase 4: Upgrade

- Add the `Upgrade` service method, API, and `thv skill upgrade` CLI.

### Phase 5: Docs

- Update `docs/arch/12-skills-system.md`, CLI docs, and swagger.

### Dependencies

None blocking. This proposal relies on existing `gitresolver` commit checkout and `ociskills.RegistryClient` digest pulls, both of which are already capable of supporting the design.

## Testing Strategy

- **Unit tests:** `lockfile` package round-trip, upsert/remove, and missing-file behavior; skillsvc hooks for project versus user scope; non-fatal lock write failures; `Sync` drift detection, unmanaged reports, and prune behavior; `Upgrade` digest changes, immutable sources, and dry-run behavior; API routes; CLI helpers.
- **E2E tests:** `thv skill install --scope project`, then `sync` on a clean tree, then `upgrade` after a catalog bump, against real GHCR artifacts.
- **Security tests:** confirm a tampered lock can only pin to references that pass existing supply-chain validation.

## Documentation

- Update `docs/arch/12-skills-system.md` with a new "Project Lock File" section, as done in the POC PR.
- Regenerate CLI docs via `task docs`.
- Add a short user-facing guide on committing `toolhive.lock.yaml` and running `sync` after pull.

## Open Questions

1. Should `sync` warn or refuse when the local SQLite state has a different digest for a skill that is in the lock, or silently reinstall to the pinned digest? The POC currently reinstalls silently.
2. Should the lock support an optional `clients` field per entry for teams that genuinely want per-client pinning, or stay strictly client-agnostic?
3. Should `upgrade` without args default to all entries, as in the current POC, or require explicit selection to avoid accidental mass upgrades in CI?
4. Where should a future manifest layer live, `toolhive.yaml` or `toolhive.skills.yaml`, if and when constraints are added, and should the lock file name change to match?

## References

- POC PR: <https://github.com/stacklok/toolhive/pull/5715>
- Research note that motivated this: <https://github.com/stacklok/notes/blob/093fc48edbc6631ef9373cc1c35581a343ac2af5/docs/joes-research/tessl-skills-package-managers.md>
- Prior art: npm `package-lock.json`, pnpm `pnpm-lock.yaml`, `Cargo.lock`, `go.sum`, `poetry.lock`
- oras-go cross-origin redirect auth guard, relevant to pinned-digest pulls: GHSA-vh4v-2xq2-g5cg

---

## RFC Lifecycle

<!-- This section is maintained by RFC reviewers -->

### Review History

| Date | Reviewer | Decision | Notes |
|------|----------|----------|-------|
| YYYY-MM-DD | @reviewer | Under Review | Initial submission |

### Implementation Tracking

| Repository | PR | Status |
|------------|----|--------|
| toolhive | [#5715](https://github.com/stacklok/toolhive/pull/5715) | POC |
