---
name: cms-site-planner
description: Turn a website brief into a phased development plan plus the project's rules and tracking documents. Use when planning a new CMS-backed website, starting a new client site, or asked for a phased development plan, implementation plan, or project setup docs for an Astro site on Cloudflare Workers — backed by Supabase, or by Cloudflare D1, R2 and Access. For greenfield projects only — not for changes inside a site that already exists.
---

# CMS site planner

Plans a new website on one of two fixed stacks, chosen per project from the brief's constraints:

- `supabase` — Astro SSR on Cloudflare Workers, Supabase for Postgres, Auth and Storage, Row-Level
  Security as the access boundary, and a custom admin in the same app.
- `cloudflare` — static Astro on Workers with a Worker for the dynamic routes, D1, R2, and a
  separate admin Worker behind Cloudflare Access; publishing rebuilds the site.

Optional capabilities are chosen per project. **This skill writes documents, never code.**

## What it emits

Six files, into the new project's directory:

```
implementation_plan.md   phases, each with a version, a rationale, its tests and a checkpoint
CLAUDE.md                session rules, commands, and the rules for this project — only those that apply
status.md                phase tracker, setup checklist, decisions, verification records, session log
deployment.md            migrations, variables, secrets, bindings, deploy order — kept current per phase
CHANGELOG.md             open [Unreleased], phase-based versioning
lessons.md               append-only record of what the build teaches
```

## Procedure

1. **Read what exists.** If the user pointed at a brief, requirement documents or a design export,
   read them first. Never re-ask something they already answer. Requirement documents stay the
   source of truth; a design export is mapped screen by screen (`references/interview.md` §0).
2. **Interview the gaps.** Work through `references/interview.md`. Ask with `AskUserQuestion`,
   batching related questions. Stop asking once the stack, every module decision and every phase
   boundary is determined.
3. **Choose the stack** from the answers, following `references/stacks.md`.
4. **Derive the phases.** Follow `references/phases.md`.
5. **Emit the six files** from `templates/`, filling every placeholder.
6. **Summarise**: the stack and the answer that decided it, which modules were selected, what you
   assumed, and what still needs the client — the last of these goes into `status.md` as well, not
   only into the chat.

## Where to read what

| Doing this | Read first |
|---|---|
| Interviewing | `references/interview.md` |
| Choosing the stack | `references/stacks.md` |
| Choosing tables, access classes, buckets | `references/schema.md` |
| Deriving phases | `references/phases.md` |
| Writing the project's `CLAUDE.md` | `references/rules-core.md`, `references/rules-<stack>.md` for the chosen stack, plus `references/rules-<module>.md` for each selected module that has one |

Stack keys: `supabase` · `cloudflare`. Module keys the interview returns: `i18n` · `media` · `blog` ·
`forms` · `gallery` · `roles` · `privacy` · `booking` · `notifications`. `blog` and `gallery`
contribute schema and phase scope only; every other module has its own rules file. A stack that was
not chosen, or a module that was not selected, contributes nothing — no tables, no phase content,
and **no rules in the generated `CLAUDE.md`**.

## Hard stops

- Ambiguity is a question, not a coin flip. If a silent brief would change the plan, ask.
- Never emit invented content as published. Placeholder copy starts unpublished, an unconfirmed
  fact is `[TBD]` — never a sample value from a design — and an empty state is safer than a
  plausible one.
- Every phase carries a version, a one-line risk rationale, the tests that must exist when it
  closes, and a runnable checkpoint.
- Rules stay generic to the stack and modules. Carry no names, values or details from another
  project into the generated files.
- Write every generated file as UTF-8 **without a BOM**, and never leave a `{{placeholder}}`,
  a `[stack: …]` or `[module: …]` marker, a `TODO` or a `…` in the output.
