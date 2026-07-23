# Campus Platform — Orchestrator

This is the **orchestrator** repo for the Campus Digitalization Platform. It runs no application code itself; it composes the platform's services and holds the cross-cutting docs and shared infra (OpenFGA authz model, Judge0 code-execution config).

The platform was split from the now-archived monorepo `Z-Zenith/Omega` into one repo per architectural container. See [`repo-split-plan.md`](repo-split-plan.md) for the split rationale and [`INTEGRATIONS.md`](INTEGRATIONS.md) for the release-compatibility matrix and per-repo versions.

## Repositories

| Repo | Container | Stack |
|---|---|---|
| [campus-backend](https://github.com/Z-Zenith/campus-backend) | Backend API (+ DB SQL) | ASP.NET Core / .NET 10 |
| [campus-ai-services](https://github.com/Z-Zenith/campus-ai-services) | AI Services | Python / FastAPI |
| [campus-student-desktop](https://github.com/Z-Zenith/campus-student-desktop) | Student Desktop App | Avalonia / .NET 10 |
| [campus-teacher-web](https://github.com/Z-Zenith/campus-teacher-web) | Teacher Web App | React + Vite |
| [campus-admin-web](https://github.com/Z-Zenith/campus-admin-web) | Admin Web App | React + Vite |
| [campus-parent-portal](https://github.com/Z-Zenith/campus-parent-portal) | Parent Portal | React + Vite |
| [campus-api-client](https://github.com/Z-Zenith/campus-api-client) | Shared API client | TypeScript |
| [campus-shared-editor-kit](https://github.com/Z-Zenith/campus-shared-editor-kit) | Shared Editor Kit | TypeScript |
| [campus-direct-messaging](https://github.com/Z-Zenith/campus-direct-messaging) | Direct Messaging | TypeScript |
| **campus-platform** (this repo) | Orchestrator / infra / docs | Docker Compose |

## Running the stack

This repo builds `campus-backend` and `campus-ai-services` from source via git submodules under `external/` (BuildKit builds from the pinned submodule commits); `postgres`, `authz` (OpenFGA), and the Judge0 code-execution services come up alongside them.

```bash
git clone --recurse-submodules https://github.com/Z-Zenith/campus-platform.git
cd campus-platform
cp .env.example .env   # set POSTGRES_PASSWORD and JWT_SIGNING_KEY
docker compose up -d
```

If you already cloned without `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

## Docs

- [Campus platform architecture.md](Campus%20platform%20architecture.md) — C4 containers, feature IDs, acceptance criteria
- [campus-platform-db-api-schema.md](campus-platform-db-api-schema.md)
- [campus-platform-starting-guide.md](campus-platform-starting-guide.md)
- [campus-platform-work-division.md](campus-platform-work-division.md)
- [INTEGRATIONS.md](INTEGRATIONS.md) — release-compatibility matrix
