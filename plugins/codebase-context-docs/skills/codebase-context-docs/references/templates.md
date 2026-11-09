# Templates and worked examples

Read this when writing the files, to see the target shape and density. These are examples from a fictional
TypeScript/Postgres service — match the *style and level of detail*, not the content.

## Contents

- [CLAUDE.md](#claudemd) — full worked example, 74 lines
- [architectural_patterns.md](#architectural_patternsmd)
- [testing_strategy.md](#testing_strategymd)
- [build_and_deploy.md](#build_and_deploymd)
- [data_model.md](#data_modelmd)
- [api_reference.md](#api_referencemd)
- [Density calibration](#density-calibration) — before/after examples of lines that earn their place

---

## CLAUDE.md

```markdown
# Harbor

Harbor is the billing and entitlements service behind our SaaS products. It owns
subscriptions, usage metering, and the entitlement checks every other service calls
before granting a feature. It is the system of record for what a customer has paid
for — correctness matters more than latency here, and most design choices trade the
latter for the former.

## Tech Stack

- TypeScript 5.4 on Node 20, ESM only
- Fastify 4 (HTTP), BullMQ (background jobs), Redis 7
- Postgres 16 via Drizzle ORM; migrations are SQL files, not generated
- pnpm workspaces (three packages, see below)
- Deployed as containers to ECS; Terraform lives in a separate repo

## Project Structure

| Path | Purpose |
|---|---|
| `packages/api/` | Fastify server. Routes, middleware, request validation. No business logic. |
| `packages/core/` | Domain logic and persistence. Everything that decides *what* happens. |
| `packages/workers/` | BullMQ consumers for metering rollups and invoice generation. |
| `packages/core/src/db/migrations/` | Hand-written SQL migrations, applied in filename order. |
| `scripts/` | One-off operational scripts. Not imported by application code. |
| `test/fixtures/` | Shared seed data. Prefer extending these over inline test setup. |

## Commands

```bash
pnpm install
pnpm dev                      # api + workers with watch, needs docker compose up first
pnpm build                    # tsc project references across all packages
pnpm test                     # vitest, full suite (requires a live Postgres)
pnpm test path/to/file.test.ts # single file
pnpm lint
pnpm typecheck
pnpm db:migrate               # applies pending SQL migrations
```

`docker compose up -d` must be running for anything that touches the database,
including most of the test suite.

## Conventions

- `packages/api` may import from `core`; `core` must never import from `api`. This
  boundary is what keeps the workers able to reuse domain logic.
- All money is integer minor units (cents) typed as `Cents`. Never floats.
- Database writes go through a repository in `core/src/repositories/`; routes never
  touch Drizzle directly.
- New endpoints must register a Zod schema — the OpenAPI spec is generated from them.
- Migrations are forward-only. There are no down migrations by design.

## Additional Documentation

Read these when the task calls for it — do not load them preemptively.

- `.claude/docs/architectural_patterns.md` — before adding a service, repository, or
  crossing a package boundary; explains the DI and error-handling conventions.
- `.claude/docs/testing_strategy.md` — before writing or debugging a test, especially
  anything involving database state or fixtures.
- `.claude/docs/data_model.md` — before adding a table, changing a schema, or writing
  a migration.
- `.claude/docs/api_reference.md` — before adding or changing an HTTP endpoint,
  auth behavior, or error response shape.
- `.claude/docs/build_and_deploy.md` — when CI fails, when adding an env var, or when
  changing anything about the container build.
```

---

## architectural_patterns.md

Longer than `CLAUDE.md` and unapologetically so — it is loaded only when relevant. Aim for one
section per pattern, each with: what the pattern is, why it exists here, where to see it, and what
to do when writing new code.

```markdown
# Architectural Patterns

## Layering: api → core, never the reverse

`packages/api` handles transport concerns only: parsing, validation, auth, serialization.
All decisions live in `packages/core`. The workers package depends on `core` too, which is
the reason for the rule — anything that leaks into `api` becomes unreachable from background jobs.

Seen in: `packages/api/src/routes/subscriptions.ts:41`, `packages/api/src/routes/usage.ts:28`,
`packages/workers/src/invoice.worker.ts:57` — all three delegate to the same
`SubscriptionService` rather than reimplementing.

When adding a feature: put the logic in a service in `core/src/services/`, then make the
route a thin caller.

## Constructor dependency injection, wired manually

There is no DI container. Services take their collaborators as constructor arguments and are
assembled once in `packages/api/src/container.ts:12` and `packages/workers/src/container.ts:9`.

`SubscriptionService` — `packages/core/src/services/subscription.service.ts:22`
`InvoiceService` — `packages/core/src/services/invoice.service.ts:19`
`EntitlementService` — `packages/core/src/services/entitlement.service.ts:15`

All three follow the same shape: repositories in, no framework types in the signature. This is
what makes them testable without a running server — see `testing_strategy.md`.

When adding a service: match the constructor shape and register it in both containers.

## Result types over thrown exceptions in the domain layer

Domain operations return `Result<T, DomainError>` rather than throwing.
`Result` — `packages/core/src/result.ts:8`. The API layer unwraps at the boundary and maps
`DomainError` to an HTTP status in `packages/api/src/error-handler.ts:33`.

Seen in `subscription.service.ts:64`, `invoice.service.ts:91`, `entitlement.service.ts:47`.

Exceptions are still thrown for programmer errors (invariant violations), not for expected
failures like "subscription not found". The distinction matters: a thrown error in the domain
layer will 500 rather than 404.

## Repository pattern over Drizzle

Query construction is confined to `packages/core/src/repositories/`. Each repository exposes
domain-shaped methods and returns domain types, never Drizzle row objects.

`SubscriptionRepository` — `packages/core/src/repositories/subscription.repository.ts:18`
`UsageRepository` — `packages/core/src/repositories/usage.repository.ts:24`

Note an inconsistency: `packages/core/src/services/legacy-export.service.ts:112` queries
Drizzle directly. It predates the repository layer (see `git log` on that file) and is the only
remaining case — follow the repositories, not this file.
```

---

## testing_strategy.md

Cover: framework and runner, where tests live and how they're named, what infrastructure they need,
fixture conventions, what's mocked vs. real, and how to run a narrow slice. The last one matters most —
an agent that can only run the full suite iterates slowly.

Include the failure modes: "if you see `ECONNREFUSED`, docker compose isn't up", "tests share a database
and must clean up in `afterEach` — see `test/setup.ts:19`".

## build_and_deploy.md

Cover: the build graph, env vars and where they're declared/validated, what CI runs on a PR and in what
order, artifacts, and the deploy path. Cite the workflow files by line. Include local-setup gotchas that
cost a newcomer an hour — a required env var with no default, a native dependency, a port conflict.

## data_model.md

Cover: the core entities and how they relate (a small text diagram beats prose), where the schema is
defined, the migration workflow and its rules, and where validation happens. Cite the schema file and
line for each entity rather than reproducing column lists, which go stale immediately.

## api_reference.md

Cover: how routes are organized and registered, request/response conventions, auth and how it's applied,
versioning, the error response shape, and pagination. Point at one exemplary route file by `file:line` as
the pattern to copy rather than enumerating every endpoint — enumerations rot, and the route files are
already the source of truth.

---

## Density calibration

The difference between a useful line and a wasted one:

| Wasted | Earns its place |
|---|---|
| "This project follows best practices for error handling." | "Domain ops return `Result<T, E>`; throwing in `core` produces a 500 instead of a 404." |
| "Tests are important — please write tests." | "`pnpm test path/to/x.test.ts` runs one file; the suite needs `docker compose up -d`." |
| "The `src/` directory contains the source code." | "`packages/api` may import `core`; `core` importing `api` breaks the workers." |
| "Uses React with hooks and modern patterns." | "State is Zustand, one store per feature — `src/stores/`; no Context providers." |
| "Follow the existing code style." | *(delete — the linter does this)* |
| "`UserService` handles user logic. ```ts\nclass UserService {…}\n```" | "`UserService` — `src/services/user.service.ts:22`" |

The test for any line: **would an agent write different code because it read this?** If not, cut it.
