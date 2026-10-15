# implementation_plan.md — template

> **To the skill.** Fill every `{{placeholder}}`. Phases come from `references/phases.md`; every
> phase needs a version, a risk rationale, its sources, its tests, a runnable checkpoint and its
> open questions. Content marked `[stack: x]` or `[module: x]` is kept only when chosen; a marker
> covers everything beneath it up to the next marker, heading or placeholder. Delete unselected
> content, every marker line, and this blockquote from the generated file.

---

# {{PROJECT_NAME}} — Implementation Plan

**Owner:** {{OWNER}} · **Created:** {{DATE}} · **Target launch:** {{LAUNCH_DATE}}

## 0. What this is

{{One paragraph: what the organisation does, who the site is for, and what "done" means.}}

Progress is tracked in [`status.md`](status.md), which mirrors these phases exactly. The rules for
changing this codebase are in [`CLAUDE.md`](CLAUDE.md); what production needs is in
[`deployment.md`](deployment.md).

### Sources of truth

| Source | Path | Role |
|---|---|---|
{{One row per requirement document and design export, with what it decides. Write "none — this
plan and the interview summary" if there are none.}}

Precedence: requirement documents win over the design. Where they conflict, the conflict is listed
in §1.2 and needs a decision before the phase shown. Phases cite requirement sections rather than
restating them.

### The stack

{{STACK_TABLE — the chosen column of references/stacks.md §1, as Layer | Choice, with the content
model filled in. Then one line: "Chosen because: <the interview answer that decided it>."}}

Free-tier limits that bind, checked on {{DATE}}: {{the rows of references/stacks.md §4 that apply}}.

### Modules selected

| Module | On? | Why |
|---|---|---|
| `i18n` | {{yes/no}} | {{locale count and primary locale, or "single locale"}} |
| `media` | {{yes/no}} | {{reason}} |
| `blog` | {{yes/no}} | {{reason}} |
| `forms` | {{yes/no}} | {{reason}} |
| `gallery` | {{yes/no}} | {{reason}} |
| `roles` | {{yes/no}} | {{reason}} |
| `privacy` | {{yes/no}} | {{the personal data held, the law that applies, the retention period}} |
| `booking` | {{yes/no}} | {{what is booked}} |
| `notifications` | {{yes/no}} | {{what is sent, and through which provider}} |

A module that is off contributes no tables, no phase content and no rules. If one is turned on
later, that is a new migration and a new phase — not a retrofit of this plan.

### Hard constraints

- Verification happens under `npm run preview`, never the dev server.
- Every rule that protects personal data, prevents a double booking or enforces a role has a test.
- Interactive controls are `min-height: 44px`. Target {{ACCESSIBILITY_TARGET}}. Every screen is
  checked at 375 px.
- {{Audience/network constraint from the interview — this sets the JS and image budgets.}}
- Nothing invented is published. Unconfirmed facts are `[TBD]`; the release check refuses them.
[stack: supabase]
- RLS is the access control. No redundant published-filters in query helpers.
- The service-role key is used by nothing.
[stack: cloudflare]
- The public build reads only the allow-listed published tables, through a sandbox.
- The admin Worker verifies the Access token on every request.
{{Plus one line per non-negotiable from the selected modules and the brief.}}

## 1. Design

### 1.1 Design system

Tokens as CSS custom properties. **No component hardcodes a hex value.**

| Token | Value | Used for |
|---|---|---|
{{DESIGN_TOKENS — colour, type scale, spacing, focus ring, light and dark. If the client supplied a
prototype, derive from it and note that the prototype is a read-only reference whose inline styles,
runtime scripts and asset loading do not ship.}}

Script-specific typography goes on `:lang()` selectors, never classes.

{{If a design export exists, keep the next three blocks; otherwise delete them.}}

**Screen → phase map.**

| Design screen | Built as | Phase |
|---|---|---|
{{Every screen in the export. Prototype viewers and explorations that are not built say "not built"
with the phase column "—".}}

**Never built:** {{prototype-only controls — state switchers, "viewing as" pickers, screen
drawers, simulated progress.}}

**Sample values in the design — never shipped:** {{every phone number, domain, opening hour, price,
registration number, name and statistic the design shows. Each is `[TBD]` in the seed until the
client confirms it.}}

### 1.2 Design and requirement conflicts

Decide each before the phase shown. Until decided, the default applies, and the decision is tracked
in [`status.md`](status.md).

| # | Conflict | Default until decided | Phase |
|---|---|---|---|
{{D1, D2, … — or "None found." }}

## 2. Data model

{{Table list by access class, from references/schema.md §2, with the draft/live decision per type
(§8) and the buckets, naming which is private.}}

## 3. Repository layout

```
{{The folders and workspaces, each with one line saying what it holds and the phase that creates
it.}}
```

## 4. Phases

Each phase lands on a version. Minor per phase, `1.0.0` at launch, patch for everything else.

| Phase | Name | Version |
|---|---|---|
{{PHASE_VERSION_TABLE — must match CLAUDE.md §6 exactly}}

{{If the owner builds everything before deploying: "Owner decision ({{DATE}}): every phase is built
and verified locally; the site is deployed once, after phase {{N}}. A phase is done when it runs
locally with lint, typecheck, tests and build passing. Every phase is still committed and pushed."}}

---

### Phase {{N}} — {{NAME}} → `{{VERSION}}`

**Goal:** {{one sentence}}

**Risk retired:** {{why this boundary exists — the question this phase answers that the next
phase would be expensive to answer}}

**Sources:** {{requirement sections and design screens this phase builds from}}

**Build:** {{Work items. Concrete enough to execute; each traceable to a rule or a requirement.}}

**Tests that must exist:** {{the automated tests this phase adds — every rule it creates that
protects personal data, prevents double booking or enforces a role has one}}

**Checkpoint:** {{a runnable command and the output that settles whether the phase is done}}

**Questions:** {{Q-numbers whose answers set this phase's final values, or "none". Unanswered
questions do not block the code; they block launch values.}}

---

{{Repeat per phase. The first phase ends at a green /api/health under preview. Launch is at 1.0.0
and is where the client's real content lands.}}

### Deferred (unscoped, unversioned)

{{Named now so it is tracked rather than discovered. An admitted gap gets closed; an unexamined
one does not.}}

## 5. Known gaps, adopted as rules

Stated here as things this build does from day one, rather than debts it repeats.

| Gap | Rule here |
|---|---|
| **Rules that protect people with no test.** Earlier builds relied on a health endpoint and manual records. | A test suite from phase 0; every phase names its tests. `/api/health` and the verification tables in [`status.md`](status.md) remain — **as well as** the suite, not instead of it. |
| **Lint and format declared but never installed.** | Do not declare a script you have not installed. Wire it up in phase 0 or delete the line. |
| **Deploy config no test ever reads.** | A config test reads the Wrangler files and checks the values that must agree; after each deploy, read what it printed. |
| **Request logs keeping tokens and addresses.** | Invocation logs off from phase 0. |
| **Work sitting uncommitted or unpushed.** | Commit each phase on its own and push it. CI that never runs checks nothing. |
| **A state document with no owner.** | [`status.md`](status.md) names its owner in its header, or it will lie — most confidently to the newest person on the project. |
[stack: supabase]
| **No caching at the data layer.** | Full SSR is right for CMS freshness and it has a bill. Memoise the shell queries per request — deferred, and tracked. |
| **Migration history drifting from the files.** | Applying SQL by hand is not applying a migration. Check `supabase migration list` before every push; repair after every hand-apply. |
[stack: cloudflare]
| **A restore nobody has run.** | Backups are tested by restoring them into a fresh local database before launch. |
{{ADDITIONAL_GAPS from the interview}}

## 6. Verification, per phase

No phase is complete on "it renders". Each closes with:

1. `npm run lint`, `npm run typecheck`, `npm test`, `npm run build` — and the phase's named tests
   present.
2. `npm run preview` — **wrangler, not the dev server.** Different runtimes; a green dev server is
   not evidence — then `curl http://127.0.0.1:8787/api/health` → 200, all checks named.
3. {{If i18n: every language exercised, including a deliberately half-translated row.}}
4. A private window used for anything about what the public sees.
5. Every new screen at 375 px{{, and as every role that can open it — if roles}}.
6. The verification table in [`status.md`](status.md) filled in — **including what was not
   checked**.
7. [`deployment.md`](deployment.md) updated if the phase added a migration, binding, variable,
   secret or outside service.
8. [`CHANGELOG.md`](CHANGELOG.md) updated, version bumped, [`status.md`](status.md) header
   refreshed, [`lessons.md`](lessons.md) appended if anything surprised you, the phase committed on
   its own and pushed.

## 7. Open questions requiring client input

{{Q1, Q2, … — everything the interview could not settle. Each with: what is blocked, the phase whose
final values it sets, and what answer unblocks it. This list is duplicated in status.md, which is
where it is maintained.}}
