# CLAUDE.md — template

> **To the skill, not to the project.** Fill every `{{placeholder}}`. Content marked
> `[module: x]` is included **only** if module `x` was selected, and content marked `[stack: x]`
> only if stack `x` was chosen — a marker covers everything beneath it up to the next marker,
> heading or placeholder. Delete unselected content entirely and every marker line. Rules come
> from `references/rules-core.md`, the chosen stack's rules file and the module rule files; carry
> each rule's *evidence*, because a rule without its cost gets argued away.
> Delete this blockquote from the generated file.

---

# CLAUDE.md

{{One paragraph: what the site is and for whom, the stack in one line, and the folders or
workspaces in the repository.}}

## Read at the start of every session

1. {{The requirement documents, if any, each with one line on what it covers. They are the source
   of truth; nothing below overrides them.}}
2. [`implementation_plan.md`](implementation_plan.md) — the scope, tests and open questions for
   each phase.
3. [`status.md`](status.md) — current phase, blockers, decisions waiting, session log.
4. [`lessons.md`](lessons.md) — what earlier sessions learned. It never overrides the documents
   above.
5. {{If a design export exists: "The design files the plan lists for the phase. They are read-only
   reference."}}

## Commands

{{From the "Commands the generated CLAUDE.md lists" table in the chosen stack's rules file, adapted
to this repository's scripts. Name what each command needs — `.dev.vars` keys with made-up local
values, a migrated and seeded local database, a stopped preview — because a command that fails for
a missing prerequisite costs a session to diagnose.}}

## Session rules

- Work on one phase per session and stay inside its scope. Anything extra goes into the notes
  column of [`status.md`](status.md).
- Write tests for every rule that protects personal data, prevents a double booking or double
  charge, or enforces a role — in the phase that creates the rule.
- End with lint, typecheck, tests and build passing; [`status.md`](status.md) updated;
  [`deployment.md`](deployment.md) updated if the phase added a migration, binding, variable, secret
  or outside service; a [`lessons.md`](lessons.md) entry if anything surprised you; the phase
  committed on its own and pushed.
- If a phase does not finish, mark it in progress in [`status.md`](status.md), list what is left,
  and finish it first next session.
- {{If a design export exists: "`<design folder>/` is read-only reference. Never ship its runtime
  scripts or its sample values; use `[TBD]` until the client confirms a value."}}
- Before any deploy, `npx wrangler whoami` must show {{the client's Cloudflare account name}}.

## Non-negotiables

{{Three to six one-line rules whose breach cannot be undone after launch, taken from the selected
modules and the brief. Examples of the kind: "No personal data in URLs, logs, page titles, browser
storage, analytics or error messages." · "Personal fields are encrypted; the key lives only in
Worker secrets." · "The public build reads only published content tables." · "Neutral wording for
<sensitive services> everywhere." · "No <banned claims under the advertising code>." Omit the
section if nothing qualifies.}}

---

Behavioural guidelines to reduce common LLM coding mistakes, followed by the rules specific to
this codebase.

**Tradeoff:** these guidelines bias toward caution over speed. For trivial tasks, use judgment.

> **Authority order.** Requirement documents > this file > [`implementation_plan.md`](implementation_plan.md).
> Where this file and the plan disagree, this one wins and the plan gets corrected.

Orientation: [`status.md`](status.md) for current state · [`implementation_plan.md`](implementation_plan.md)
for phase scope · [`deployment.md`](deployment.md) for what production needs ·
[`lessons.md`](lessons.md) for what we have learned · [`CHANGELOG.md`](CHANGELOG.md) for what
shipped and why.

---

## 1. Think before coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

- State assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity first

**Minimum code that solves the problem. Nothing speculative.** No features beyond what was asked,
no abstractions for single-use code, no configurability nobody requested, no error handling for
impossible scenarios. If you write 200 lines and it could be 50, rewrite it.

## 3. Surgical changes

**Touch only what you must. Clean up only your own mess.** Don't improve adjacent code, don't
refactor what isn't broken, match existing style. Remove imports and variables *your* change made
unused; leave pre-existing dead code alone and mention it instead.

The test: every changed line traces directly to the request.

If files you did not touch changed while you worked, leave them alone and check `git status` before
committing, so you commit only your own changes.

## 4. Goal-driven execution

Transform tasks into verifiable goals. "Add validation" → "write the failing cases, then make them
pass". For multi-step work, state the plan as steps each with its own check.

---

# {{PROJECT_NAME}} project rules

Everything below extends the four guidelines above. Where they conflict, these win — they encode
constraints specific to this codebase.

## 5. Project conventions

{{RULES — assembled from references/rules-core.md, the chosen stack's rules file, and the module
rule files for each selected module. Keep the Rule / Why / Evidence shape. Do not include a rule
for a module that was not selected or a stack that was not chosen. Include rules-core.md §14b only
if the interview named an advertising code.}}

### The stack

{{STACK_TABLE — the chosen column of references/stacks.md §1, as a two-column table (Layer |
Choice), with the content model filled in: "JSONB per translatable field, {{LOCALES}}", "translation
tables, {{LOCALES}}", or "plain text columns, single locale".}}

---

## 6. Version and changelog

**Every change updates [`CHANGELOG.md`](CHANGELOG.md). No exceptions for "small" changes.**

### Versioning is phase-based

`package.json` version tracks the plan, not semver-for-APIs — this is a website with no downstream
consumers, so "breaking change" has no useful meaning here.

| Bump | When | Example |
|---|---|---|
| **Minor** `0.N.0` | A phase completes | Phase 0 done → `0.1.0` |
| **Patch** `0.N.x` | Anything else: fixes, docs, tooling, content | `0.1.1` |
| **Major** `1.0.0` | Public launch (Phase {{LAUNCH_PHASE}}) | — |

**Phase → version, for this project:**

| Phase | Version |
|---|---|
{{PHASE_VERSION_TABLE — one row per phase, matching implementation_plan.md exactly}}

**Current: `{{CURRENT_VERSION}}`.**

> This line is the one piece of *state* in a rules file, and it is exactly the line that goes
> stale. [`status.md`](status.md) is authoritative; if the two disagree, this one is wrong.

### The workflow

1. Make the change.
2. **Add an entry under `## [Unreleased]` in [`CHANGELOG.md`](CHANGELOG.md)**, in the right group
   (Added / Changed / Fixed / Removed / Security / Notes). Write it for the person reading it
   later, not as a restatement of the diff.
3. **When a phase completes or a fix ships:** rename `[Unreleased]` to the new version with
   today's date, bump `package.json`, run `npm install --package-lock-only` so the lockfile
   carries the same number, and open a fresh empty `[Unreleased]`.
4. Update the [`status.md`](status.md) header (last updated, commit, phase row), add a
   verification-record row and a session-log row.
5. **If the change added a migration, binding, variable, secret or outside service, update
   [`deployment.md`](deployment.md)** in the same commit.
6. **If you learned something, add it to [`lessons.md`](lessons.md).** If the learning hardens
   into a rule, **promote it into this file** and fill in that entry's `Promoted:` line.

### What belongs in the changelog

User-visible changes and anything a future maintainer would be surprised by. Not every refactor —
`git log` records those better.

Do record: bugs that were subtle, decisions that diverged from the plan, and anything
security-relevant (the access boundary, credentials, the staff gate, personal data). **The "why"
matters more than the "what"** — the diff already has the what.

**Write the failures down too.** An honest changelog is a debugging tool; a promotional one is not.

### What belongs in lessons.md

[`lessons.md`](lessons.md) is **append-only** and deliberately overlaps the changelog: the
changelog records **what shipped**, lessons records **what was learned** — including from work
that never shipped.

Entry format:

```
### YYYY-MM-DD · <short title>
**Trigger:** what actually happened.
**Learning:** the generalisable point.
**Rule:** the one-sentence imperative — or "none yet".
**Promoted:** link to the CLAUDE.md section, once it gets there.
```

**The trigger is the load-bearing part.** A learning stated without the thing that produced it
gets argued away by the next person who finds it inconvenient — usually correctly, because a claim
with no cost attached looks like superstition. Quantify where you can: "184 KB to animate six
numbers" ends a discussion; "React felt heavy" starts one.

Two sections: **A — Collaboration** (corrections, preferences, instructions misread, approaches
confirmed right) and **B — Technical**. A lesson that never gets promoted is still worth keeping;
most of them are context, not policy.

### Commit and push at the boundary you would want to roll back to

A deploy pipeline that triggers on push does nothing for work that was never pushed, and CI checks
nothing it never ran. Commit each phase on its own, and push it — even when the owner deploys only
at the end.

---

## 7. Known blind spots

Written down because a convenience you have not named is a convenience you will trust.

- **Tests build their own environment, so they never see the deploy config.** The config test
  (`rules-core.md` §18) covers the values that must agree; everything else in the Wrangler files is
  checked by reading what the deploy printed.
- **What the tests do not cover is listed in [`status.md`](status.md)** under paths never exercised.
  A green suite is evidence about the paths it runs, and about nothing else.

[stack: supabase]
- **A signed-in editor sees unpublished drafts on the public site**, because the same cookie-bound
  client serves both. Kept deliberately — it is free draft preview — but it means **"it looks fine
  to me" is not evidence about what visitors see.** Check in a private window.
- **There is no caching at the data layer.** Every request hits the database, and the base layout
  runs its chrome queries before the page's own. Deferred and tracked — not discovered later.

[stack: cloudflare]
- **Published content is only as fresh as the last build.** A publish whose rebuild failed leaves
  the site on old content with nothing visibly wrong; the admin shows the publish job's state.
- **The admin's draft preview and the public site are different renderers.** Check the published
  result on the built site.
- **A spent D1 daily quota stops every dynamic route until midnight UTC**; the content pages stay up,
  which makes it easy to miss.

{{ADDITIONAL_BLIND_SPOTS — anything the interview surfaced, e.g. an unproven locale count, an
unprovisioned account, a module whose interaction was designed but never run, a stack whose
production cutover is unproven}}
