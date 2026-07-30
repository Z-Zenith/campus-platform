# Spec: Per-College Database Migration (multi-tenant router)

## Status

**Draft — Phase 1 (Specify) of spec-driven-development. Not approved. No implementation has
started.** This document exists to get the shape of the change agreed before any code, migration,
or schema edit is written, per this workspace's contract-change rule (DB schema) and the scale of
what's being proposed here.

## Objective

Today the platform is a single shared Postgres database (`campus`), with every tenant-scoped table
carrying a `college_id` column and every controller enforcing row-level isolation via
`ICollegeScopeService`/`CollegeScopeService`. Confirmed with the user (2026-07-30): this is
changing to **one dedicated Postgres database per college**, each reachable via its own
connection string, with a single running `backend-api` process resolving which database to talk
to on a per-request basis ("one app, many DBs" — not one full deployment stack per college).

**Why** (inferred, not yet confirmed — flagged as an assumption below): stronger data isolation
between colleges than row-level `college_id` scoping provides — a bug in a `WHERE college_id = ?`
clause today is a cross-tenant data leak; a bug in tenant-DB routing under this design fails
closed (wrong connection, not wrong rows) or is contained to picking the wrong *whole database*,
not exposing one college's rows inside another's response.

## Assumptions I'm making — correct me now

1. **`college_id` columns are NOT being removed from the schema in this migration.** Each
   per-college database still only ever holds that one college's rows, so `college_id` becomes a
   constant, always-the-same-value column — dead weight, but harmless. Removing it would mean
   touching every table and every controller's existing scoping logic, which conflicts with all
   ten currently-open, unmerged PRs (`campus-backend` #50/#52/#53/#54, `campus-admin-web`
   #23/#24/#26, `campus-student-desktop` #20, `campus-api-client` #10, `campus-platform` #20) and
   multiplies this migration's blast radius by roughly an order of magnitude for close to zero
   functional benefit. Proposed: **keep the column, keep `ICollegeScopeService` as-is, revisit
   removal later as an optional, separate cleanup** once the router itself is stable.
2. **The `College` table's role changes but the table doesn't disappear.** It still exists inside
   each per-college database (id, name, etc. — useful for display, config, future per-college
   settings) but stops being the thing that makes row-level isolation work. A *new*, separate
   piece of infrastructure — a small "directory" store outside any single college's database —
   is what maps a college to its connection string. This directory is new; it does not exist in
   any form today. **Confirmed by the user (2026-07-30): this directory is a JSON file/config,
   an array of `{ name, id, connectionString }` entries** — not a database table, matching the
   "not a new database" default already proposed below. `id` is the stable key a request resolves
   against (see Decision 2); `name` is the human-readable label a login screen would show in a
   picker.
3. **Session validation has the same pre-auth problem as login.** `UserSession` rows live inside
   the per-college database, so validating a bearer token/session on *every* authenticated request
   also needs to know which database to check before it can run a single query — not just at
   login. Proposed: the JWT itself carries the college identity (see Tenant Resolution below), so
   this is solved once, not twice.
4. **This migration lands as its own foundational change, sequenced *before* the ten open PRs
   merge, not after.** See Decision 1 below — flagging this as the assumption I'd default to
   absent a correction, not something already decided.
5. **Every existing college's data starts in exactly one database.** There is, as far as I've
   found, exactly one college's worth of real/seed data in the current shared `campus` database
   today (single-college dev/demo state) — I have not verified this by querying a live database
   (none was running this session). If there is already more than one college's data live in the
   shared DB, this migration also needs a one-time data-splitting step (export college A's rows to
   database A, college B's rows to database B) that isn't scoped below. Flagging rather than
   assuming either way.

## Decision 1 (blocks everything else): sequencing against the ten open PRs

This determines the entire phase ordering below — get this settled first.

| Option | What it means | Cost |
|---|---|---|
| **A. This migration first** | The ten open PRs (Community/Events redesign, Phase 9, the enum cleanup, the doc catch-up, the admin-web/desktop/api-client companions) all get rebased onto whatever `Program.cs`/`AppDbContext` construction changes this introduces. | Every one of those PRs needs a rebase + retest once this lands, since `AppDbContext`'s registration path changes from `AddDbContext` (built once at startup) to something resolved per-request. Delays all ten PRs further. |
| **B. This migration after** | The ten open PRs merge first, on the current single-shared-DB assumption. This migration then lands as its own follow-up on top of a settled `main`. | No rework for already-shipped-but-unmerged work. But every one of the ten PRs' controllers/tests still assume a single ambient `AppDbContext` — this migration's Phase 2/3 (tenant resolution, per-request `DbContext` construction) has to retrofit *all* of them, including brand-new code (Clubs/ClassroomDiscussions/Events approval) that was just written against the old assumption. |

**No default recommended here** — this is a project-management call (how many PRs are actually
close to merge-ready vs. how much rework either ordering causes), not a technical one. Needs your
call before Phase 2 planning starts.

## Decision 2: tenant resolution — how does a request know which database to use?

**Confirmed by the user (2026-07-30): the login-field default below is accepted, specifically as
a searchable dropdown** (not a plain `<select>`/free-text field) populated from `GET /colleges`'s
`{id, name}` list — every one of SDA/TWA/AWA/PRT's login screens needs this same searchable-
picker treatment, not just a bare dropdown, since the college list is expected to grow past what a
plain dropdown stays usable for.

This is the design's spine — every other piece (login, session validation, every controller)
depends on it, and it's the one place client apps (SDA/TWA/AWA/PRT) must change.

**The hard constraint**: `LoginRequest` today is `(Identifier, Password, TotpCode, DeviceInfo)` —
no college identifier anywhere, on any of the four client apps. Login can't query a `users` table
that doesn't exist yet in any single database until it knows which database to query. Session
validation (`UserSession`) has the identical problem on every subsequent request.

**Recommended default** (concrete, so you have something to accept or reject rather than a bare
menu):

1. `LoginRequest` gains one new field: `CollegeId` — the `id` from the tenant directory's
   `{ name, id, connectionString }` entries (confirmed shape, see assumption 2 above). Every
   client's login screen gains one field/dropdown for it (populated from a small, unauthenticated
   `GET /colleges` returning just `{ id, name }` pairs — never `connectionString` — so a login
   screen can show a human-readable picker rather than asking someone to type a raw id). This is
   the only client-visible surface change.
2. The Backend API process, at startup, loads the **tenant directory** JSON (college id → name +
   connection string) from its own configuration — e.g. a mounted file path in config, read once
   at startup and re-read on a timer or restart. Adding a college is an ops action (append one
   entry, restart or hot-reload), not a schema migration. Where this JSON file lives/how it's
   deployed (baked into the image, mounted volume, config-service fetch) isn't decided yet — a
   Work Item below, not blocking the shape of the directory itself.
3. `POST /auth/login` resolves `CollegeId` against the directory, opens a connection to *that*
   college's database, validates identifier/password/TOTP against it, and — on success — stamps
   the resolved college id as a claim in the issued JWT.
4. Every subsequent authenticated request carries that claim. A tenant-resolution middleware reads
   it, resolves the same directory, and hands the right connection string down to wherever
   `AppDbContext` gets constructed for that request (see Risk section — this replaces
   `AddDbContext`'s current once-at-startup registration).

**Alternative considered, not recommended**: subdomain-per-college (`iare.campus.example.com`).
Cleaner in principle (no login-screen change) but needs real DNS/certificate provisioning per
college and doesn't solve anything for the Student Desktop App (a native app, not a browser tab
resolving a hostname the same way) — SDA would still need some equivalent of a college-code field
or a first-run "which college" picker. Named here so you can pick it explicitly if you prefer it
over the login-field default.

**Also considered, not recommended**: a tenant HTTP header the client sets on every call instead
of a JWT claim. Rejected because it means every one of SDA/TWA/AWA/PRT's HTTP call sites needs to
thread an extra header through, versus the JWT-claim approach where only the login call site
changes and everything downstream is transparent.

## Decision 3 (smaller, but real): does `campus-api-client`'s login contract change?

If Decision 2's default is accepted, `LoginRequest`/the login call in `campus-api-client` gains
`collegeCode`. That package already has one unmerged version bump pending (task #29 —
`EventType`/Holiday fields, PR #10, blocked on being cut as a tag). This migration would be a
**second**, unrelated reason to bump that same package before task #29's consumer
(`campus-admin-web`'s Holiday UI) can even be built — flagging the stacking rather than treating
it as a coincidence to resolve silently.

## Work items (once Decision 1 and 2 are settled)

1. **Tenant directory** — confirmed: a JSON file, an array of `{ name, id, connectionString }`
   entries, loaded at Backend API startup (not a new database). Still open: where the file lives
   in each deployment (baked into the image vs. a mounted volume vs. fetched from somewhere at
   startup), and whether adding a college requires a process restart or can hot-reload.
2. **Per-request `AppDbContext` construction** — replace the current single `AddDbContext(...)`
   registration (built once, connection string fixed for the process's lifetime) with a
   per-request-resolved one. Needs care: `Program.cs`'s existing `.MapEnum<AccountType>()` …
   `.MapEnum<WhitelistRequestStatus>()` chain (11 Postgres enum mappings as of this session, see
   `campus-backend` PR #54) must be applied identically to every dynamically-constructed
   `NpgsqlDataSource`/options instance — missing one silently breaks every query touching that
   enum column, and the EF InMemory provider used by this repo's whole test suite won't catch that
   class of mistake (already bit this session once, on unrelated default-value behavior — see the
   Events-redesign phase's InMemory-vs-Postgres note). Also: cache one `NpgsqlDataSource`/
   connection pool **per tenant**, not one per request — building a fresh pool per call leaks
   connections under load.
3. **Login + session validation** — `AuthController`'s login endpoint resolves the tenant *before*
   constructing the request-scoped `DbContext`; token validation middleware does the same from the
   JWT claim for every other endpoint.
4. **Schema bootstrap & fan-out tooling** — `db/init/*.sql` today runs exactly once, via
   Postgres's `docker-entrypoint-initdb.d`, against a single fresh volume (confirmed by reading
   `campus-platform/docker-compose.yml`'s `postgres` service this session). That mechanism cannot
   provision a new college's database mid-life, and cannot apply a schema change to N *existing*
   college databases at once. `campus-backend/MIGRATIONS.md`'s current policy ("migrations don't
   exist yet, `db/init/*.sql` is the one source of truth, applied once on first boot") does not
   survive this change as written — needs its own real migration-fan-out story: a
   provision-new-college-database script/command, and an apply-schema-change-to-every-tenant path
   (real EF Core migrations, run in a loop over the tenant directory, rather than continuing to
   rely on first-boot-only init scripts).
5. **Docker Compose / deployment topology** — `campus-platform/docker-compose.yml`'s `postgres`
   service currently models exactly one database. Needs a decision on what "one app, many DBs"
   looks like operationally in Compose terms (N Postgres containers with a fixed, small,
   known-at-compose-time set of colleges for dev/demo purposes, vs. Postgres services assumed to
   live outside Compose entirely in real deployments, with Compose only ever running one for local
   dev). Not designed yet — needs Decision 1/2 settled first.
6. **Architecture doc** — Section 7's Container View ("Database: PostgreSQL... System of record")
   and Section 9's opening paragraph (which explicitly argues *for* a relationship-graph tenant
   boundary specifically to avoid "a `college_id` column threaded through every query by hand")
   both describe the single-shared-DB model as the deliberate, decided architecture. Per this
   doc's own established pattern (see the 2026-07-09 changelog entry marking the OpenFGA
   enforcement description "aspirational/historical, not current behavior" rather than silently
   rewriting it), this migration should get the same treatment once approved: a dated changelog
   entry + updated Section 7/9 text, not a silent rewrite of the existing prose.

## Explicitly out of scope for this migration (flag if you disagree)

- Removing `college_id` from any table (assumption 1 above).
- A public self-serve college-signup flow — Section 16's existing open question already frames
  multi-college onboarding as a manual, Ruthvik-run step; this migration doesn't change that,
  it just changes what "onboard a college" produces (a new database + a new tenant-directory
  entry, instead of a new row in a shared `colleges` table).
- Splitting any *existing* mixed-tenant data across databases (assumption 5 — flagged as
  possibly-not-applicable, not designed here).

## Open questions for you

1. **Decision 1** (sequencing) — see table above.
2. **Decision 2** (tenant resolution) — directory shape confirmed (`{name, id, connectionString}`
   JSON), login mechanism confirmed (`CollegeId` field via a searchable dropdown on every client's
   login screen, populated from a public `GET /colleges`). Still open: where does the JSON file
   live per deployment (baked into the image / mounted volume / fetched at startup)?
3. **Decision 3** (api-client stacking) — fine to let this migration be a second reason to cut a
   new `campus-api-client` tag, on top of task #29's existing one?
4. Roughly how many colleges is this meant to scale to, and on what timeline? (Doesn't change the
   design above, but changes how much to invest in the provisioning-tooling piece vs. a
   manual-for-now script.)
