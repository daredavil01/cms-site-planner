# Changelog

All notable changes to the `cms-site-planner` plugin.

## [0.2.0] — 2026-10-15

Lessons from a full build on a second stack, folded back in as generic rules.

### Added
- **A second stack, `cloudflare`** — static Astro on Workers, D1, R2, and an admin Worker behind
  Cloudflare Access, publishing by rebuild. `references/stacks.md` decides between it and `supabase`
  from the brief's constraints (zero cost, public sign-in, freshness, per-visitor pages) rather than
  by asking "which stack?", and lists the free-tier limits that bind on each.
  `references/rules-cloudflare.md` carries that stack's rules: the build reads an allow-list of
  published tables through a sandbox, a D1 batch is the only transaction, the D1 limits and their
  workarounds, a real local D1 per test, Access tokens verified in the Worker, restore order, two
  cron triggers per Worker.
- **Three modules:** `privacy` (field encryption bound to the row, retention jobs, view logging, an
  exposure checklist reviewed before launch), `booking` (slot locks under a unique index, engine
  before screens, a concurrency test where every pair overlaps, private manage links) and
  `notifications` (own-domain senders, a message log, atomic claims for scheduled sends, signed
  webhooks). `forms` and `roles` gained rules files of their own.
- **`templates/deployment.md`**, a sixth emitted file: migrations, variables, secrets, bindings,
  outside services, deploy order and post-deploy checks, updated by every phase that adds one.
- **Core rules** for lessons that bind on any project: test the rules that protect people from phase
  0 (§7); unconfirmed facts are `[TBD]` and the release check refuses them (§14a); regulated
  wording as a word-list test (§14b, only when an advertising code applies); every screen at 375 px
  (§15); test the deploy config and read what the deploy printed (§18); request logs are a data
  store (§19); security headers per response path (§20); checks against the built output (§21);
  scripted edits keep line endings (§22).
- **Planning inputs:** design exports mapped screen by screen with their sample values listed as
  never-ship; design/requirement conflicts as numbered decisions with defaults; numbered open
  questions that block launch values, not code; deploy cadence and session size as interview
  questions.

### Changed
- **`rules-core.md` is now stack-independent.** RLS, the service-role key, the Supabase key scan, the
  `security definer` RPC rule, `getUser()` and the cookie jar, and migration drift/repair moved to
  `references/rules-supabase.md` with their evidence intact. Core §4 (access control), §5 (secrets),
  §8 (auth) and §12 (migrations) keep the general rule and point to the stack file for mechanics.
- **Phases** carry their sources, the tests that must exist, and the open questions that block their
  values, and are sized for one working session. New cut rules: admin foundation before admin
  screens, data protection with the first public-write surface, engine before screens, and a
  quality-pass phase with a security and privacy review. Planned post-launch phases are versioned
  `1.1.0`+; "build locally, deploy once" is a supported variant. A third worked example covers the
  `cloudflare` stack.
- **`CLAUDE.md` template** opens with what to read each session, the commands, session rules and
  non-negotiables. "There is no test suite" is no longer an accepted blind spot.
- **`status.md` template** gains a setup checklist separating *confirmed by the owner* from *verified
  here*, a decisions table, a blockers column, and a session log.
- **`rules-i18n.md`** covers the static stack's choice between path prefixes and a browser swap, the
  layout-shift fixes for the swap, fallback formatting, pseudo-translation testing and review before
  publish. The N≥3 caveat records that four languages ran as a row of plain links.
- **`rules-media.md`**: what the per-image ceiling protects differs by stack; the browser optimiser
  without a client framework; server re-checks; documents (PDF compression, script scanning,
  serving through a route); responsive-image fallbacks and lazy images moving anchors.
- **`schema.md`**: access classes drive the export allow-list on `cloudflare`; drafts and published
  copies; `CHECK` constraints that pass on `NULL`; seeds and answer files; feature switches.

### Notes
- **Generic by rule.** The evidence behind every new rule is written as a failure pattern, not as a
  story about the project it came from; the skill's hard stops now forbid carrying another project's
  names, values or details into generated files.
- **What is still unproven is said.** The `cloudflare` stack has a full build and a preview deploy
  but no production cutover yet; a Supabase test suite is designed, not run. Both caveats travel into
  each generated `status.md`.

## [0.1.0] — 2026-08-22

### Added
- Initial release. A skill that interviews for a website brief and emits five project documents:
  `implementation_plan.md`, `CLAUDE.md`, `status.md`, `CHANGELOG.md` and `lessons.md`.
- Six reference files carrying the distilled rules in Rule / Why / Evidence form: `rules-core.md`
  (17 always-applicable rules), `rules-i18n.md`, `rules-media.md`, `schema.md`, `phases.md` and
  `interview.md`.
- Module system: `i18n`, `media`, `blog`, `forms`, `gallery`, `roles`. A module that is off
  contributes no tables, no phase content and no rules.

### Fixed
- **"Storage columns hold paths, never URLs" moved from `rules-media.md` into `rules-core.md` as
  §12a**, found by running the plugin's own negative test. With the `media` module off, that rule
  vanished — but it binds on any project with a storage bucket, and the test project had two path
  columns and no rule governing them. `rules-core.md` §12a now also points at the egress
  arithmetic in `rules-media.md` §4, because a dozen fixed photographs are still served over the
  same metered transfer as a library.

### Notes
- **Phases are derived from scope, not templated.** Four constraints keep derivation from becoming
  improvisation: a foundation phase first ending at a green health endpoint, a launch phase last at
  `1.0.0`, a version + risk rationale + runnable checkpoint on every phase, and an explicit
  phase-to-version table emitted in two places. `phases.md` carries two worked examples as a
  calibration target.
- **`SKILL.md` is a router, not a rulebook** — 57 lines, one code block, no rule text and no
  project-specific names. Everything substantive lives one file deeper, so the always-loaded body
  stays small.
- **The `i18n` rules generalise to N locales, and say where the evidence stops.** Everything in
  that file is proven at two languages; the toggle-becomes-a-picker and ordered-fallback-chain
  consequences of three or more are designed but unrun, and the skill carries that caveat into
  each generated `status.md` rather than hiding it.
