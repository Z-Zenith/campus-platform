# Repo-wide TODO / unfinished-work inventory

Compiled 2026-07-30 by sweeping every `campus-*` repo's code comments (TODO/FIXME/HACK/XXX,
`NotImplementedException`/`NotImplementedError`, disclosed-gap prose) and every markdown doc
(README, CLAUDE.md, architecture doc, migration/integration notes) for unfinished, deferred, or
explicitly-disclosed-incomplete work. This is a point-in-time inventory, not a live backlog —
re-sweep periodically rather than trusting this file once significant work has landed. Where an
item already has an open PR addressing it, that PR is named; items with no PR are genuinely
untracked. Items are grouped by repo, with a cross-repo section first for things that span more
than one.

---

## Cross-repo / architectural

- **Multi-tenant per-college database migration** — spec drafted (`docs/multi-tenant-db-migration-spec.md`,
  PR [#21](https://github.com/Z-Zenith/campus-platform/pull/21), draft, not approved). Two decisions
  block everything else: sequencing against the ten other currently-open PRs, and where the tenant-
  directory JSON file lives per deployment. No implementation has started.
- **Architecture doc says AIS-06 is "Won't priority/future," but it was actually built this session** —
  `Campus platform architecture.md` line 140 and line 249 still read "Won't"/"no stack decision... not
  built this round," yet AIS-06 (LLM-based syllabus extraction) shipped as `campus-ai-services` PR
  [#18](https://github.com/Z-Zenith/campus-ai-services/pull/18) and `campus-backend` PR
  [#51](https://github.com/Z-Zenith/campus-backend/pull/51) (unmerged, but real, tested code — not
  vaporware). The doc has drifted from actual repo state; needs a priority-reclassification pass once
  those PRs merge, not left silently inconsistent.
- **`campus-teacher-web`'s Community page will break once the Community redesign merges** — already
  logged in the architecture doc as TWA-05a (added in PR [#20](https://github.com/Z-Zenith/campus-platform/pull/20)),
  but unfixed. `campus-teacher-web/src/lib/api.ts` and `CommunityPage.tsx`/`MaterialsPage.tsx` call
  `/groups`, `/groups/mine`, `/groups/{id}/posts` and use `GroupDto`/`GroupPostDto` — all of which stop
  existing once `campus-backend` PR [#52](https://github.com/Z-Zenith/campus-backend/pull/52) (Clubs/
  ClassroomDiscussions/StaffGroups split) merges. This is the same class of break already fixed once
  for `campus-student-desktop` (PR #20 there) — `campus-teacher-web` still needs the equivalent fix.
  Track 2 territory per this workspace's CLAUDE.md.
- **OpenFGA (Authorization Service) is designed but never wired into enforcement** — `services/authz/model.fga`
  exists and documents the intended ReBAC shape, but `campus-backend`'s `PermissionService.cs` enforces
  every permission check directly against Postgres tables (`role_bindings`/`roles`/`permission_grants`)
  instead. Already resolved as an explicit decision (architecture doc changelog, 2026-07-09, issue #76):
  `model.fga` is reference-only, "revisit only if a concrete driver materializes (e.g. true multi-college
  ReBAC needs outgrow the flat table model)" — worth noting the multi-tenant DB migration above could
  itself become exactly that driver, if per-college isolation needs grow past what `college_id`-in-a-
  now-single-tenant-DB provides.
- **Judge0 (Code Execution Service) sandboxing not fully verified** — `campus-platform/services/code-execution/judge0.conf`
  flags `judge0-workers`' fork-safety as "not-yet-fully-verified in this Docker Desktop/WSL2 environment,"
  with queued submissions able to sit at "In Queue" indefinitely even after a `FORK_PER_JOB=false` fix.
  Also: no concrete feature in the architecture doc currently calls for live code execution via Judge0
  (SEK-01 uses a different, already-implemented `DockerCodeRunner` path per `campus-backend/CLAUDE.md`) —
  Judge0 itself has no real consumer yet (`repo-split-plan.md`'s deferred-items table).
- **Cross-repo shared design system extraction** — `repo-split-plan.md` names this as a deferred,
  untracked item (no issue filed): pull Tailwind/shadcn primitives currently duplicated across
  `campus-teacher-web`/`campus-admin-web`/`campus-parent-portal` into a new `campus-design-system`
  repo. Flagged as a decision point if ever prioritized, not started.
- **Generated API clients (OpenAPI-codegen) instead of hand-mirrored DTOs** — `repo-split-plan.md`
  deferred item, untracked: pick a generator (e.g. NSwag for C#→TS), wire to the OpenAPI snapshot,
  decide how generated types coexist with/replace `campus-api-client`'s hand-written ones. This is the
  root cause of the whole "keep `campus-api-client` in sync by hand" pattern that's produced repeated
  version-bump/contract-drift friction this session (task #29's blocked Holiday UI, `AdminEventDto`
  workarounds, etc.).
- **No continuous publishing pipeline for versioned cross-repo packages** — `INTEGRATIONS.md`'s "known
  gaps at cutover" section: `campus-api-client`/SEK/DMS tags are cut manually today, not CI-driven.

---

## Architecture doc — Section 16 "Open Questions" (verbatim, still open)

| Question | Owner | Status |
|---|---|---|
| Multi-college onboarding: decided as a manual step Ruthvik runs today, with internal automation tooling preferred over a public self-serve signup flow — the automation itself still needs to be built and scoped | Ruthvik | Open |
| **Data subject rights feature gap**: DPDP grants students/parents rights to access, correct, and request erasure of their data — no feature in Section 3 currently handles this as a formal request/response workflow; needs scoping (who fulfills it — Admin? IT?) before it can be a real feature | Ruthvik | Open |
| **Consent/privacy notice at admission**: decide whether this is in scope for this platform or handled separately by the institution's admission process | Ruthvik | Open |
| Grievance/Data Protection Officer contact point — decide whether this is a Section 3 feature (a grievance form) or purely an institutional/offline process outside this system's scope | Ruthvik | Open |

Related, not yet a scoped feature: `campus-platform-db-api-schema.md`'s "Open Items" section notes
the data-subject-rights workflow has no backing table yet — add one once the feature above is scoped,
don't build ahead of the open question being resolved.

Also unbuilt, explicitly marked non-goals for now (not bugs, just named future scope): voice-recognition
attendance, Google Calendar sync, WhatsApp/auto-call fee reminders (architecture doc line ~49).

---

## campus-backend

- `Controllers/CalendarController.cs:302` — "today" calculation uses `DateTime.UtcNow`, which can shift
  the weekly calendar boundary for non-UTC colleges near midnight. Real fix needs a `College.TimeZone`
  schema column. Tracked as a follow-up, not blocking any current PR.
- `Services/DockerCodeRunner.cs:19` — assumes `docker` is reachable from wherever `backend-api`'s process
  runs (true for bare `dotnet run`). If `backend-api` is ever containerized, needs the host's Docker
  socket bind-mounted in — a real security tradeoff (host-root-equivalent access), not currently wired
  up, deliberately left as a future decision rather than silently assumed.
- `Services/CopyleaksClient.cs:28`, `Services/AiServicesClient.cs:34` — AIS-02/05 (Copyleaks/Pangram) are
  external-API-only; no live sandbox/credentials available in this dev environment to verify against.
  Out of scope for this environment, not a code gap — flagging so a reviewer with real credentials
  smoke-tests both before relying on them in production.
- `Controllers/WebhooksController.cs:27` — Copyleaks webhook auth uses a static Bearer token; a dynamic
  mTLS-thumbprint-pinning scheme would be stronger but is infra-level and out of scope for this code.
- `Controllers/FeesController.cs:63,74` — Payment Gateway integration is a stub (PRT-03/AWA-04 flow
  through a simulated, synchronously-confirmed gateway) — the real external Payment Gateway integration
  itself is explicitly out of scope per the architecture doc (Section 4, external system).
- `Controllers/FeesController.cs:136` — no background-job runner (`IHostedService`) exists anywhere in
  this codebase yet; `SendReminders` and other periodic-style requirements (AIS-03's copy-check, SDA-13's
  grace-period alert) are invoked on an externally-managed schedule instead of a first-party scheduler.
- `Controllers/BrowsingController.cs:13`, `Controllers/TelemetryController.cs:76` — whitelisted-browser/
  telemetry/AI-services-forwarding endpoints are stubbed just enough to keep the shared API contract
  complete; full implementation is Track 2 territory, not yet built. Telemetry forwarding to AI Services
  is explicitly "best-effort" — raw telemetry persists safely even if AI Services is unreachable.

## campus-admin-web

- `src/components/layout/Header.tsx:78-88` — the header search input is disabled with placeholder text
  "Search (coming soon)." Purely decorative (Fluent-style command-bar visual pattern) — no app-wide
  search backend exists to wire it to.

## campus-teacher-web

- `src/pages/AssignmentDetailPage.tsx:276-277` — AIS-05 (AI-generated-content likelihood) explicitly
  shows "Pending — Copyleaks scan not yet complete" when the score is undefined; only AIS-02 (internet
  plagiarism) is live in this deployment today. (Backend/frontend PRs already exist for AIS-05 — see
  "Already in flight" below — this is the state on `main` before those merge.)
- See the cross-repo section above: this repo's `/groups/*` calls will break once the backend Community
  redesign merges (TWA-05a).

## campus-parent-portal

No unfinished-work markers found — complete, production-ready state as of this sweep.

## campus-student-desktop

- `CLAUDE.md:46-49` — building SEK/DMS host bundles (`npm run build:host`) before `dotnet build` is a
  manual dev prerequisite, not yet wired into a cross-toolchain CI step. Missing `dist/host` silently
  yields empty `SekHost`/`DmsHost` bundles rather than a build error (Content globs don't fail on no
  matches) — a real footgun for anyone who forgets the manual step.
- `Services/AssignmentAutoSubmitService.cs:22-24,49` — the SDA-11 auto-submit-on-exit service is fully
  implemented and wired for a caller, but no assignment-editing view exists in this repo yet, so its
  exit/focus-loss handlers are currently no-ops with nothing to submit. Waiting on that view's own
  implementation, not a bug in this service.

## campus-ai-services

No unfinished-work markers found — all implemented features (AIS-01 through AIS-07) have full test
coverage and no disclosed gaps in code or docs.

## campus-shared-editor-kit (SEK)

- `README.md:14` — SEK-04 (built-in image search) is an interface-only stub in `main`; no runtime
  implementation merged yet. (PRs [#10](https://github.com/Z-Zenith/campus-shared-editor-kit/pull/10)
  and [#15](https://github.com/Z-Zenith/campus-shared-editor-kit/pull/15) [draft] exist — see "Already
  in flight" below.)
- `README.md:20` — same `npm run build:host`-before-`dotnet build` manual-step gap as
  `campus-student-desktop`'s CLAUDE.md above; not yet CI-enforced.
- `README.md:82-83` — component-level (React-rendering) tests for SEK-01/02/04 are deferred until a DOM-
  testing dependency is added to the project.
- `CLAUDE.md:26-27` — cross-repo host-bundle distribution mechanism to Student Desktop is described as
  currently unresolved (documented in `campus-student-desktop`'s own CLAUDE.md, referenced from here).

## campus-direct-messaging (DMS)

- `CLAUDE.md:25-26` — same unresolved cross-repo host-bundle distribution mechanism to Student Desktop
  as SEK above.

## campus-api-client

No unfinished-work markers found. Version is `0.1.0`; no `CHANGELOG.md` exists (`RELEASE.md` documents
the tagging convention but doesn't track unreleased changes) — worth adding once the package sees more
churn, given how many currently-open PRs across other repos are blocked on a specific tag being cut
(task #29's Holiday UI; the multi-tenant migration's likely `collegeCode` field; PR
[#10](https://github.com/Z-Zenith/campus-api-client/pull/10)'s own pending `EventType` bump).

## campus-platform

- `services/authz/README.md:1-12` — status banner confirms OpenFGA is "not currently invoked" (see
  cross-repo section above).
- `docker-compose.yml:209-213` — `judge0-workers`' internal-network-only exposure is explicitly called
  out as one piece of a multi-part sandboxing security rationale; removing any one mitigation without
  the others is flagged as dangerous. Not a gap, just a fragility worth knowing before touching that
  service.
- `campus-platform-db-api-schema.md:366,446` — `GET /submissions/{id}/ai-detection` (AIS-05) and
  `POST /telemetry` (SDA-25) are both marked "not yet implemented" in this doc; both now have real,
  unmerged implementations in `campus-backend`/`campus-ai-services`/`campus-teacher-web` (see "Already
  in flight" below) — this doc line is stale relative to those PRs' existence, not relative to `main`.
- `campus-platform-db-api-schema.md:471-478` — open item: `DocumentDescriptor.title` in SEK is optional
  and not backed by any `documents` column — unresolved whether title should become a stored column or
  stay derived from `file_url`/filename. Whichever wins, the SEK interface should be revisited.
- `repo-split-plan.md:79-82` — the actual `file:` → `github:<org>/campus-<pkg>#0.1.0` swap in consuming
  apps' `package.json`/`.csproj` files was deliberately deferred to immediately after each shared-lib
  repo's own split, not done pre-split — noted here only as a sequencing fact already accounted for, not
  a currently-open gap (the swap has since happened per the pinned-tag references seen across repos).

---

## Already in flight (open PRs addressing items above — not unaddressed, just unmerged)

So the items above aren't mistaken for having zero attention: as of this sweep, `gh pr list` shows real,
open work already targeting several of them —

- AIS-05 (AI-content detection): `campus-backend` [#16](https://github.com/Z-Zenith/campus-backend/pull/16)
  (Pangram integration), `campus-teacher-web` [#13](https://github.com/Z-Zenith/campus-teacher-web/pull/13) (UI).
- SEK-04 (image search): `campus-shared-editor-kit` [#10](https://github.com/Z-Zenith/campus-shared-editor-kit/pull/10),
  [#15](https://github.com/Z-Zenith/campus-shared-editor-kit/pull/15) (draft, WebView bridge wiring),
  `campus-backend` [#16](https://github.com/Z-Zenith/campus-backend/pull/16), `campus-teacher-web`
  [#10](https://github.com/Z-Zenith/campus-teacher-web/pull/10).
- SDA-25 telemetry / API-02 class-group scheduler: `campus-backend`
  [#17](https://github.com/Z-Zenith/campus-backend/pull/17).
- AIS-06 syllabus extraction: `campus-ai-services` [#18](https://github.com/Z-Zenith/campus-ai-services/pull/18),
  `campus-backend` [#51](https://github.com/Z-Zenith/campus-backend/pull/51),
  `campus-admin-web` [#24](https://github.com/Z-Zenith/campus-admin-web/pull/24).
- Multi-tenant DB migration spec: `campus-platform` [#21](https://github.com/Z-Zenith/campus-platform/pull/21) (draft).
- Section 9 doc catch-up (class_teacher/Community/Events): `campus-platform` [#20](https://github.com/Z-Zenith/campus-platform/pull/20).

This list is not exhaustive of every open PR (there are ~20-40 open per repo, many dependency-bump
chores) — only PRs that map directly to a gap named above.
