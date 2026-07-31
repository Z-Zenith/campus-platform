# Repo & version compatibility matrix

`campus-platform` is the orchestrator repo — it does not run any app itself. It pulls pinned
versions of every other component. This table tracks which tagged version of each repo is
known-compatible with which others.

## Repos

| Repo | Tier | Source path in original `Omega` |
|---|---|---|
| [campus-admin-web](https://github.com/Z-Zenith/campus-admin-web) | App | `apps/admin-web/` |
| [campus-teacher-web](https://github.com/Z-Zenith/campus-teacher-web) | App | `apps/teacher-web/` |
| [campus-parent-portal](https://github.com/Z-Zenith/campus-parent-portal) | App | `apps/parent-portal/` |
| [campus-student-desktop](https://github.com/Z-Zenith/campus-student-desktop) | App | `apps/student-desktop/` |
| [campus-ai-services](https://github.com/Z-Zenith/campus-ai-services) | App | `services/ai-services/` |
| [campus-backend](https://github.com/Z-Zenith/campus-backend) | App | `services/backend-api/` + `db/` |
| [campus-api-client](https://github.com/Z-Zenith/campus-api-client) | Shared lib | `packages/api-client/` |
| [campus-shared-editor-kit](https://github.com/Z-Zenith/campus-shared-editor-kit) | Shared lib | `packages/shared-editor-kit/` |
| [campus-direct-messaging](https://github.com/Z-Zenith/campus-direct-messaging) | Shared lib | `packages/direct-messaging/` |
| campus-platform (this repo) | Orchestrator | root files, `docs/`, `services/authz/`, `services/code-execution/` |

## Current compatible set

| Component | Version |
|---|---|
| campus-backend | `main`, current as of this update |
| campus-api-client | `0.1.1` — supersedes `0.1.0`, adds the mailmap consolidation and a configurable API base URL (`setApiBaseUrl`) |
| campus-shared-editor-kit | `0.2.1` (npm) / `host-0.2.1` (host bundle) — supersedes `0.1.0`/`host-0.1.0` |
| campus-direct-messaging | `0.1.1` (npm) / `host-0.1.1` (host bundle) — supersedes `0.1.0`/`host-0.1.0` |
| campus-teacher-web | on an active feature branch (`feature/twa-fixes-track1-track2-bundle`, PR #18) as of this update — **still pins `@campus/api-client`/`@campus/shared-editor-kit`/`@campus/direct-messaging` at the old `0.1.0` tags**; re-pin to `0.1.1`/`0.2.1`/`0.1.1` once that PR merges and the repo is back on `main` |
| campus-admin-web | on the same active feature branch (PR #9) as of this update — **still pins `@campus/api-client` at `0.1.0`**; re-pin to `0.1.1` once merged |
| campus-parent-portal | `main` + PR #9 (`chore/repin-api-client-0.1.1`) — re-pins `@campus/api-client` to `0.1.1`, `npm run build`/`npm test` green |
| campus-student-desktop | `main` + PR #14 (`chore/repin-sek-dms-host-0.2.1-0.1.1`) — host bundles resolved via git submodules `external/shared-editor-kit` and `external/direct-messaging`, bumped to `host-0.2.1`/`host-0.1.1`; `dotnet build`/`dotnet test` green. See that repo's `CLAUDE.md` |
| campus-ai-services | `main`, current as of this update |

## Bootstrap: cloning the full set

```bash
for repo in campus-admin-web campus-teacher-web campus-parent-portal campus-student-desktop \
            campus-ai-services campus-backend campus-api-client campus-shared-editor-kit \
            campus-direct-messaging; do
  git clone "https://github.com/Z-Zenith/$repo.git"
done
```

Then `docker compose up -d` from this repo brings up Postgres, OpenFGA (`services/authz/`),
the Judge0-based Code Execution Service (`services/code-execution/`), and — via `build:` contexts
pointing at the sibling clones above, or pinned images once a publishing pipeline exists —
`campus-backend` and `campus-ai-services`.

## Known gaps at cutover

- ~~campus-student-desktop's SEK/DMS WebView host bundle integration has no resolved cross-repo
  distribution mechanism~~ — resolved: both are git submodules under `external/`, pinned to
  `host-0.1.0`. Still a manual dev prerequisite (`npm run build:host` in each submodule before
  `dotnet build`) rather than CI-wired, same as pre-split — not a new gap, just pointed at the
  right location now.
- No continuous publishing pipeline yet (see `repo-split-plan.md`'s non-goals) — the "current
  compatible set" above is a point-in-time record, not automatically enforced by CI.
- `campus-teacher-web`/`campus-admin-web` re-pins to `0.1.1`/`0.2.1` are pending on
  `feature/twa-fixes-track1-track2-bundle` (PRs #18/#9) merging first — deliberately not
  touched while that branch is under active development, to avoid colliding with in-flight
  work. Re-pin once merged.
- `campus-shared-editor-kit` has already moved past `0.2.1`/`host-0.2.1` — `0.3.0`/`host-0.8.0`
  are cut and include the SEK-04 image-search restore, the Notes rich-text rewrite, and the
  Coding app's VS Code-style panel/terminal work. No consumer (`campus-teacher-web`,
  `campus-student-desktop`) has adopted either yet, so `0.2.1`/`host-0.2.1` stays the "current
  compatible set" above by this doc's own known-compatible-not-just-latest definition — flagged
  here so the next refresh doesn't have to rediscover the gap.
