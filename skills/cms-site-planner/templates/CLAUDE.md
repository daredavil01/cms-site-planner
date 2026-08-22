# CLAUDE.md — template

> **To the skill, not to the project.** Fill every `{{placeholder}}`. Sections marked
> `[module: x]` are included **only** if module `x` was selected — delete the whole section
> otherwise, heading included. Rules come from `references/rules-core.md` and the module rule
> files; carry each rule's *evidence*, because a rule without its cost gets argued away.
> Delete this blockquote from the generated file.

---

# CLAUDE.md

Behavioural guidelines to reduce common LLM coding mistakes, followed by the rules specific to
this codebase.

**Tradeoff:** these guidelines bias toward caution over speed. For trivial tasks, use judgment.

> **Authority order.** This file > [`implementation_plan.md`](implementation_plan.md). Where the
> two disagree, this one wins and the other gets corrected.

Orientation: [`status.md`](status.md) for current state · [`implementation_plan.md`](implementation_plan.md)
for phase scope · [`lessons.md`](lessons.md) for what we have learned · [`CHANGELOG.md`](CHANGELOG.md)
for what shipped and why.

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

## 4. Goal-driven execution

Transform tasks into verifiable goals. "Add validation" → "write the failing cases, then make them
pass". For multi-step work, state the plan as steps each with its own check.

---

# {{PROJECT_NAME}} project rules

Everything below extends the four guidelines above. Where they conflict, these win — they encode
constraints specific to this codebase.

## 5. Project conventions

{{RULES — assembled from references/rules-core.md, plus the module rule files for each selected
module. Keep the Rule / Why / Evidence shape. Do not include a rule for a module that was not
selected.}}

### The stack

| Layer | Choice |
|---|---|
| Framework | Astro, with {{FRAMEWORK}} islands for genuinely interactive UI |
| Rendering | On-demand SSR (`output: 'server'`) |
| Runtime | Cloudflare Workers (**workerd**), via `@astrojs/cloudflare` |
| Data | Supabase — Postgres + Auth + Storage, with Row-Level Security as the access boundary |
| CMS | A custom admin in this app, behind server-side auth |
| Content | {{CONTENT_MODEL — "JSONB per translatable field, {{LOCALES}}" or "plain text columns, single locale"}} |

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
4. Update the [`status.md`](status.md) header (last updated, commit, phase row) and add a
   verification-record row.
5. **If you learned something, add it to [`lessons.md`](lessons.md).** If the learning hardens
   into a rule, **promote it into this file** and fill in that entry's `Promoted:` line.

### What belongs in the changelog

User-visible changes and anything a future maintainer would be surprised by. Not every refactor —
`git log` records those better.

Do record: bugs that were subtle, decisions that diverged from the plan, and anything
security-relevant (RLS, the privileged-credential boundary, the admin gate). **The "why" matters
more than the "what"** — the diff already has the what.

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

### Commit at the boundary you would want to roll back to

A deploy pipeline that triggers on push does nothing for work that was never pushed. Commit at
each phase boundary, and push.

---

## 7. Known blind spots

Written down because a convenience you have not named is a convenience you will trust.

- **A signed-in editor sees unpublished drafts on the public site**, because the same cookie-bound
  client serves both. Kept deliberately — it is free draft preview — but it means **"it looks fine
  to me" is not evidence about what visitors see.** Check in a private window.
- **There is no test suite.** `/api/health` and the verification tables in [`status.md`](status.md)
  stand in for one. They are genuinely better than nothing and they do not stop being a substitute.
  Name the untested paths where someone will see them.
- **There is no caching at the data layer.** Every request hits the database, and the base layout
  runs its chrome queries before the page's own. Deferred and tracked — not discovered later.
{{ADDITIONAL_BLIND_SPOTS — anything the interview surfaced, e.g. an unproven locale count, an
unprovisioned account, a module whose interaction was designed but never run}}
