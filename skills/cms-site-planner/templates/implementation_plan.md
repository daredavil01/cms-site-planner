# implementation_plan.md — template

> **To the skill.** Fill every `{{placeholder}}`. Phases come from `references/phases.md`; every
> phase needs a version, a risk rationale and a runnable checkpoint. Delete this blockquote from
> the generated file.

---

# {{PROJECT_NAME}} — Implementation Plan

**Owner:** {{OWNER}} · **Created:** {{DATE}} · **Target launch:** {{LAUNCH_DATE}}

## 0. What this is

{{One paragraph: what the organisation does, who the site is for, and what "done" means.}}

Progress is tracked in [`status.md`](status.md), which mirrors these phases exactly. The rules for
changing this codebase are in [`CLAUDE.md`](CLAUDE.md).

### The stack

| Layer | Choice |
|---|---|
| Framework | Astro, with {{FRAMEWORK}} islands for genuinely interactive UI |
| Rendering | On-demand SSR (`output: 'server'`) |
| Runtime | Cloudflare Workers (**workerd**), via `@astrojs/cloudflare` |
| Data | Supabase — Postgres + Auth + Storage, RLS as the access boundary |
| CMS | A custom admin in this app, behind server-side auth |
| Content | {{CONTENT_MODEL}} |

### Modules selected

| Module | On? | Why |
|---|---|---|
| `i18n` | {{yes/no}} | {{locale count and primary locale, or "single locale"}} |
| `media` | {{yes/no}} | {{reason}} |
| `blog` | {{yes/no}} | {{reason}} |
| `forms` | {{yes/no}} | {{reason}} |
| `gallery` | {{yes/no}} | {{reason}} |
| `roles` | {{yes/no}} | {{reason}} |

A module that is off contributes no tables, no phase content and no rules. If one is turned on
later, that is a new migration and a new phase — not a retrofit of this plan.

### Hard constraints

- Verification happens under `npm run preview`, never the dev server.
- RLS is the access control. No redundant published-filters in query helpers.
- The service-role key is used by nothing.
- Interactive controls are `min-height: 44px`. Target {{ACCESSIBILITY_TARGET}}.
- {{Audience/network constraint from the interview — this sets the JS and image budgets.}}
- Nothing invented is published. Placeholder content is `published = false`.

## 1. Design system

Tokens as CSS custom properties. **No component hardcodes a hex value.**

| Token | Value | Used for |
|---|---|---|
{{DESIGN_TOKENS — colour, type scale, spacing. If the client supplied a prototype, derive from
it and note that the prototype is a read-only reference whose inline styles do not ship.}}

Script-specific typography goes on `:lang()` selectors, never classes.

## 2. Data model

{{Table list by access class, from references/schema.md. Name the buckets and which is private.}}

## 3. Phases

Each phase lands on a version. Minor per phase, `1.0.0` at launch, patch for everything else.

| Phase | Name | Version |
|---|---|---|
{{PHASE_VERSION_TABLE — must match CLAUDE.md §6 exactly}}

---

### Phase {{N}} — {{NAME}} → `{{VERSION}}`

**Goal:** {{one sentence}}

**Risk retired:** {{why this boundary exists — the question this phase answers that the next
phase would be expensive to answer}}

{{Work items. Concrete enough to execute; each traceable to a rule or a requirement.}}

**Checkpoint:** {{a runnable command and the output that settles whether the phase is done}}

---

{{Repeat per phase. The first phase ends at a green /api/health under preview. The last phase is
launch at 1.0.0 and is where the client's real content lands.}}

### Deferred (post-launch, unversioned)

{{Named now so it is tracked rather than discovered. An admitted gap gets closed; an unexamined
one does not.}}

## 4. Known gaps, adopted as rules

Stated here as things this build does from day one, rather than debts it repeats.

| Gap | Rule here |
|---|---|
| **No test suite.** A smoke endpoint plus manual records stand in for one. | `/api/health` and the per-phase verification tables in [`status.md`](status.md) are the day-one baseline — **and they do not stop being a substitute.** Name the untested paths explicitly where someone will see them. |
| **Lint and format declared but never installed.** | Do not declare a script you have not installed. Wire it up in phase 0 or delete the line. |
| **No caching at the data layer.** | Full SSR is right for CMS freshness and it has a bill. Memoise the shell queries per request — deferred, and tracked. |
| **Migration history drifting from the files.** | Applying SQL by hand is not applying a migration. Check `supabase migration list` before every push; repair after every hand-apply. |
| **Work sitting uncommitted.** | Commit and push at each phase boundary. A pipeline that triggers on push does nothing for work never pushed. |
| **A state document with no owner.** | [`status.md`](status.md) names its owner in its header, or it will lie — most confidently to the newest person on the project. |
{{ADDITIONAL_GAPS from the interview}}

## 5. Verification, per phase

No phase is complete on "it renders". Each closes with:

1. `npm run build && npm run preview` — **wrangler, not the dev server.** Different runtimes; a
   green dev server is not evidence.
2. `curl http://127.0.0.1:8787/api/health` → 200, all checks named.
3. {{If i18n: every language exercised, including a deliberately half-translated row.}}
4. A private window used for anything about what the public sees.
5. The verification table in [`status.md`](status.md) filled in — **including what was not
   checked**.
6. [`CHANGELOG.md`](CHANGELOG.md) updated, version bumped, [`status.md`](status.md) header
   refreshed, [`lessons.md`](lessons.md) appended.

## 6. Open items requiring client input

{{Everything the interview could not settle. Each with: what is blocked, and what answer unblocks
it. This list is duplicated in status.md, which is where it is maintained.}}
