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
| campus-backend | `main` @ cutover |
| campus-api-client | `0.1.0` |
| campus-shared-editor-kit | `0.1.0` (npm) / `host-0.1.0` (host bundle) |
| campus-direct-messaging | `0.1.0` (npm) / `host-0.1.0` (host bundle) |
| campus-teacher-web | `main` @ cutover (pins the above three at `0.1.0`) |
| campus-admin-web | `main` @ cutover (pins campus-api-client at `0.1.0`) |
| campus-parent-portal | `main` @ cutover (pins campus-api-client at `0.1.0`) |
| campus-student-desktop | `main` (post-cutover fix) — host bundles resolved via git submodules `external/shared-editor-kit` and `external/direct-messaging`, both pinned to `host-0.1.0`; see that repo's `CLAUDE.md` |
| campus-ai-services | `main` @ cutover |

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
