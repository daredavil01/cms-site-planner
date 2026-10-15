# cms-site-planner

A Claude Code plugin that turns a website brief into a phased development plan and the documents
that keep the build honest afterwards.

It plans one of **two fixed stacks**, chosen from the brief's constraints:

- **`supabase`** — Astro SSR on Cloudflare Workers, Supabase for Postgres/Auth/Storage, RLS as the
  access boundary, and a custom admin in the same app.
- **`cloudflare`** — static Astro on Cloudflare Workers with a Worker for the dynamic routes, D1,
  R2, and a separate admin Worker behind Cloudflare Access; publishing rebuilds the site. For briefs
  that must cost nothing, where a free-tier database that pauses after a quiet week is not an option.

The rules it carries are specific enough to check because each stack is fixed — that is the trade,
and it is deliberate. A stack that was not chosen contributes no rules.

**It writes documents, not code.**

## What it produces

| File | Holds |
|---|---|
| `implementation_plan.md` | Phases, each with a version, a risk rationale, its sources, the tests that must exist and a runnable checkpoint |
| `CLAUDE.md` | What to read each session, the commands, session rules, non-negotiables, and the rules for this project — only the ones the stack and modules make true |
| `status.md` | Phase tracker, setup checklist, decisions, verification records, paths never exercised, open questions, session log |
| `deployment.md` | Migrations, variables, secrets, bindings, outside services, deploy order and post-deploy checks — updated by every phase that adds one |
| `CHANGELOG.md` | Open `[Unreleased]`, phase-based versioning |
| `lessons.md` | Append-only record of what the build teaches |

## Modules

The core is always planned. These are asked about and only planned if chosen:

`i18n` (multi-locale content) · `media` (ID-referenced media library) · `blog` · `forms`
(public-write surfaces) · `gallery` · `roles` (more than one kind of editor) · `privacy` (sensitive
personal data: encryption, retention, view logging) · `booking` (time that must not be
double-booked) · `notifications` (email, reminders, staff alerts).

A module that is off contributes no tables, no phase content, and **no rules** — a monolingual
site never receives a translation rule it will not use.

## Install

```bash
claude plugin marketplace add /path/to/cms-site-planner
```

Then install `cms-site-planner` from that marketplace. Validate a local checkout with:

```bash
claude plugin validate /path/to/cms-site-planner
```

## Use

In an empty directory for the new site:

> Plan a new CMS-backed website. The brief is in `./brief.md`.

or, with no brief at all:

> Plan a new website for a community health charity — I'll answer questions.

The skill reads whatever you point it at — a brief, requirement documents, a design export — asks
only about the gaps, and writes the six files.

## Where the rules came from

Two production sites on the `supabase` stack, and one full build on the `cloudflare` stack with a
test suite, deployed to preview. Every rule in `skills/cms-site-planner/references/` is stated as
**Rule · Why · Evidence**, and the evidence is the load-bearing part — a rule without the failure it
prevents gets argued away by the next person who finds it inconvenient. The evidence is written
generically: no project's names, values or data travel into another project's plan.

Where a rule has not run in production, the reference says so, and the skill carries that caveat
into the generated `status.md`.

## Scope

Greenfield only. It does not audit or re-plan a project that already has code.
