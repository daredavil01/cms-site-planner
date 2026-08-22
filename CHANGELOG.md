# Changelog

All notable changes to the `cms-site-planner` plugin.

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
