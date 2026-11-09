---
name: codebase-context-docs
description: Analyzes a codebase and writes a lean CLAUDE.md (under 150 lines) plus supporting .claude/docs/ reference files using progressive disclosure — architectural_patterns.md, testing_strategy.md, build_and_deploy.md, data_model.md, api_reference.md. Use this skill whenever the user asks to generate, create, write, refresh, or improve a CLAUDE.md, AGENTS.md, project context file, onboarding doc, or "docs so Claude understands this repo"; whenever they say their CLAUDE.md is bloated, stale, or too long; whenever they ask to document a codebase's architecture, patterns, or conventions for an AI agent; and whenever they run /init or ask to "index", "onboard onto", "get up to speed on", or "map out" an unfamiliar repository. Prefer this skill over ad-hoc exploration for any of those requests, even when the user does not name CLAUDE.md explicitly.
---

# Codebase Context Docs

Generate a compact `CLAUDE.md` at the repo root plus a set of deep-dive files under `.claude/docs/`.

The goal is not "document the project." It is: **make a future agent effective on its first turn while spending as few tokens as possible.** `CLAUDE.md` is loaded into context on every single turn, forever. Every line in it is a tax. A line earns its place only if it changes what an agent does on a task it hasn't seen yet. Everything else belongs in `.claude/docs/`, loaded only when relevant.

That trade-off drives every rule below.

## Workflow

Follow this order. Writing `CLAUDE.md` first is the most common failure — it produces a plausible-sounding file full of guesses, and the docs then get written to match the guesses instead of the code.

1. **Survey** the repo mechanically (commands below).
2. **Read** the entry points, config, and a representative slice of source.
3. **Write `.claude/docs/*.md`** — the detail lives here.
4. **Write `CLAUDE.md` last** — it is an index over what you just learned.
5. **Verify** — line count, every command, every `file:line`.

### Step 1 — Survey

Run these before reading anything. They cost little and prevent wrong guesses about size, language mix, and tooling.

```bash
# Shape and scale
git ls-files | wc -l
git ls-files | sed 's|/[^/]*$||' | sort | uniq -c | sort -rn | head -40
git ls-files | grep -oE '\.[a-z0-9]+$' | sort | uniq -c | sort -rn | head -20

# Tooling and entry points (read every file this finds)
ls -a
git ls-files | grep -iE '(package\.json|pyproject\.toml|setup\.py|requirements.*\.txt|Cargo\.toml|go\.mod|pom\.xml|build\.gradle|Gemfile|composer\.json|pubspec\.yaml|Makefile|justfile|Taskfile|docker-compose|Dockerfile)'
git ls-files '.github/workflows/*' '.gitlab-ci.yml' '.circleci/*'

# Existing docs — read these, they encode intent you cannot infer from code
git ls-files | grep -iE '(readme|contributing|architecture|adr|docs/)' | head -30

# Recent activity shows which parts of the repo are actually alive
git log --oneline -15
git log --pretty=format: --name-only -n 200 | sort | uniq -c | sort -rn | head -20
```

Note anything the survey *fails* to find. A repo with no CI config and no test directory is a real finding, not a gap to paper over.

### Step 2 — Read

Read, in this order:

- Manifest files (`package.json`, `pyproject.toml`, …) — end to end, including the scripts/tasks section. This is where real commands live.
- CI workflow files — the commands CI runs are the commands that actually work.
- `README.md` and any `CONTRIBUTING.md` / `docs/` — for the *why*, which code cannot tell you.
- The application entry point(s), then 5–15 representative source files spread across the main directories.

Use parallel subagents for the source-reading pass when the repo is large and subagents are available — one per major directory, each reporting back what that directory does, what patterns it uses, and the `file:line` anchors for those patterns. This is faster and keeps raw file dumps out of the main context.

While reading, keep a running list of candidate patterns with the exact `path/to/file.ts:142` where you saw each one. You will need these anchors later, and going back to find them is expensive.

### Step 3 — Write `.claude/docs/`

Create only the files the codebase actually supports. An empty or padded doc is worse than a missing one, because it teaches the agent to distrust the whole set.

| File | Create when | Contains |
|---|---|---|
| `architectural_patterns.md` | Always (unless the repo is trivially small) | Recurring structural decisions: DI, state management, error handling, module boundaries, API design conventions, layering rules |
| `testing_strategy.md` | Tests exist | Frameworks, where tests live, fixture/factory conventions, how to run one test vs. the suite, what is mocked vs. real |
| `build_and_deploy.md` | Build tooling or CI is non-obvious | Build pipeline, env vars and where they're declared, artifacts, release/deploy flow, local dev setup gotchas |
| `data_model.md` | Persistent data, schemas, or migrations exist | Core entities and relationships, schema/migration location and workflow, validation layer |
| `api_reference.md` | The project exposes or consumes a significant API | Route/endpoint organization, request/response conventions, auth, versioning, error shape |

You may add a domain-specific file when a codebase has a large subsystem that doesn't fit these (`realtime_sync.md`, `billing.md`). Prefer a small number of substantial files over many thin ones.

#### The multi-file evidence rule

This is the rule that separates a useful patterns doc from a hallucinated one.

**Document a pattern only if you have seen it in at least three separate files, and cite at least two of them.** A single clever abstraction is trivia. A convention repeated across the codebase is what a new agent must follow to write code that fits in.

When you find something that looks like a pattern but appears once, either drop it or grep to confirm it generalizes:

```bash
rg -n 'useAppDispatch' --stats           # how widespread is it?
rg -l 'implements Repository' | head     # which files share this shape?
```

If a convention is inconsistently applied — half the codebase does it one way, half another — say so explicitly and note which is newer (`git log` on the files tells you). An agent that knows a codebase is mid-migration writes better code than one that assumes uniformity.

#### Reference by anchor, not by snippet

Write `See src/services/base.py:34` rather than pasting the class. Snippets go stale the moment someone edits the file, and a stale snippet actively misleads — whereas a stale line number is obviously worth re-checking. It also keeps the docs short enough to be read.

Include the symbol name alongside the line so the anchor survives small edits: `PaymentProcessor.execute — src/billing/processor.py:88`.

#### Leave code style alone

The user runs linters and formatters. Do not document indentation, quote style, line length, import ordering, or naming casing. Those are enforced mechanically; restating them in an always-loaded file spends context to duplicate a tool. Do document conventions a linter *cannot* enforce — which layer may import which, when to add a migration, where a new route registers itself.

### Step 4 — Write `CLAUDE.md`

Under 150 lines, hard. Use this structure:

```markdown
# <Project Name>

<2-4 sentences: what this project is and who/what it serves. The WHY —
the problem it solves. This is the part that cannot be inferred from code,
so it is the part most worth writing carefully.>

## Tech Stack

<Compact list: language + version, framework, database, key infrastructure.
Only what changes how code gets written. Skip the transitive dependency dump.>

## Project Structure

<Table or list of top-level directories, one line each, saying what belongs
there and — where non-obvious — what does not.>

## Commands

<Only verified, universally useful commands: install, dev, build, test,
test-a-single-file, lint, typecheck. Note the package manager if it is not
the default one for the ecosystem.>

## Conventions

<3-8 lines, only rules that apply repo-wide and that a linter cannot catch.
If a rule is specific to one subsystem, it belongs in a docs file instead.>

## Additional Documentation

Read these when the task calls for it — do not load them preemptively.

- `.claude/docs/architectural_patterns.md` — <when to read it>
- `.claude/docs/testing_strategy.md` — <when to read it>
```

Two things make the "Additional Documentation" section work or fail:

- Each entry says **when to read it**, not what it contains. "Read before adding an endpoint or changing a response shape" routes an agent correctly; "documents the API" does not.
- The list is the *only* place specialized detail is referenced. If you find yourself explaining the testing setup in `CLAUDE.md` *and* in `testing_strategy.md`, delete it from `CLAUDE.md`.

See `references/templates.md` for a filled-in example of `CLAUDE.md` and each docs file.

### Step 5 — Verify

Do not skip this. Every failure mode of this task is a confident-sounding falsehood, and every one is cheap to catch.

```bash
wc -l CLAUDE.md                          # must be < 150
```

Then:

- **Run every command you wrote down**, or at minimum confirm each exists in the manifest / CI config. A `npm test` in `CLAUDE.md` that isn't in `package.json` will waste an agent's time on every future session. If you cannot run a command, mark it as read-from-source rather than verified.
- **Spot-check every `file:line`** — open a few and confirm the cited symbol is there.
- **Re-read `CLAUDE.md` cold**, as an agent seeing this repo for the first time. Delete any line that doesn't change what you'd do. Generic advice ("write clean code", "handle errors appropriately") is the usual offender.
- Report the final line count and the list of files created.

If `CLAUDE.md` is over 150 lines, cut rather than compress — move a whole section into a docs file and leave one index line behind. Squeezing prose to fit produces a dense file nobody parses.

## Working with an existing CLAUDE.md

If one already exists, don't overwrite blind. Read it first, then verify it against the current code: commands that no longer exist, directories that were renamed, patterns that were migrated away from. Preserve any hand-written instruction that encodes a human preference you cannot derive from the code ("never touch the vendored SDK", "ask before adding a dependency") — those are the highest-value lines in the file and the easiest to destroy. Show the user what you're removing and why before you cut anything substantial.
