# Stacks

The skill plans one of **two** fixed stacks. Fixed, because a rule is only checkable when the stack
under it is known — "verify the policy under an anonymous session" means nothing on a database with
no policies. Two, because the third site this skill learned from could not run on the first stack
at its budget, and planning it on the wrong one would have been worse than not planning it.

A stack that was not chosen contributes nothing: no rules, no commands, no blind spots. A D1 project
never receives an RLS rule.

---

## 1. The two stacks

| Layer | `supabase` | `cloudflare` |
|---|---|---|
| Framework | Astro, on-demand SSR (`output: 'server'`) | Astro, static output, plus a Worker entry for the dynamic routes |
| Runtime | Cloudflare Workers (**workerd**), via `@astrojs/cloudflare` | Cloudflare Workers Static Assets + one site Worker (**workerd**) |
| Data | Supabase Postgres, Row-Level Security as the access boundary | Cloudflare D1 (SQLite); the Worker's route guards and the build's export allow-list are the access boundary |
| Files | Supabase Storage | Cloudflare R2; images copied into the build, documents streamed through the Worker |
| Staff sign-in | Supabase Auth, gated in middleware | Cloudflare Access in front of a separate admin Worker, which verifies the Access token itself |
| Admin | In the same Astro app | A separate Worker (Hono, server-rendered JSX, plain HTML forms) on `admin.<domain>` |
| Publishing | Live on save — public pages read the database per request | Publish marks drafts published and triggers a rebuild (deploy hook); live in 1–3 minutes |
| Scheduled jobs | — (add only if a module needs them) | Workers Cron triggers |

The rules for each live in `rules-supabase.md` and `rules-cloudflare.md`. Everything in
`rules-core.md` binds on both.

## 2. Choosing

Ask the questions that decide it; do not ask "which stack?". A client cannot answer that, and the
answer is in their constraints.

| If the brief says… | Choose | Because |
|---|---|---|
| It must cost nothing, allow commercial use, and traffic may be quiet for a week at a time | `cloudflare` | Supabase Free pauses a project after **7 days of low activity**. A paused database takes every SSR page down with it, not only the forms. |
| Members of the public sign in (accounts, saved data, members' areas) | `supabase` | Cloudflare Access is a staff gate (free up to 50 users), not a customer login. |
| An editor's save must be visible at once | `supabase` | The static stack publishes by rebuilding. |
| Public pages vary per visitor or per request | `supabase` | Static pages are the same for everyone; anything per-visitor is a Worker route. |
| Content pages must stay up when the database is down or over quota | `cloudflare` | Content pages are static HTML. A spent D1 daily quota stops forms, booking and the admin until midnight UTC, never the content. |
| Search or reporting needs more than SQLite | `supabase` | Postgres. |

*Default:* `supabase`, the stack with production evidence — **unless** the budget answer is "free" and
traffic is low or bursty, in which case `cloudflare`. Either way, state the choice and the answer
that decided it in the summary and in `status.md`.

*Evidence:* a zero-cost, commercial-use brief compared five stacks and rejected Supabase Free for
the pause alone. Nothing in this skill's first release would have surfaced that; it asked about
budget only to size the egress ceiling.

## 3. What each stack's evidence is

Carry this into the generated `status.md` under paths never exercised. A stack is not proven by
the planner's confidence in it.

- **`supabase`** — two production sites. Neither had an automated test suite; the test approach in
  `rules-supabase.md` §8 is designed, not run.
- **`cloudflare`** — one full build with a test suite, deployed to a preview environment.
  **Production cutover had not happened** when these rules were written, so anything that only
  production exercises (real Access sign-ins at volume, the D1 daily quota, email deliverability) is
  unproven.

## 4. Free-tier limits that bind

Checked October 2026. **Re-check at kick-off** and write the date into the plan — these change, and
a limit you remembered is a limit you did not check.

| Stack | Limit | What it changes in the plan |
|---|---|---|
| both | Workers: 100k requests/day for Worker code; static asset requests free | Static content costs nothing; every API call and SSR page counts |
| both | **5 cron triggers per account**, and every preview Worker's triggers count | Plan at most two triggers per Worker (e.g. every 15 minutes, nightly) and dispatch jobs inside them |
| `supabase` | Project pauses after 7 days of low activity | A keep-alive is a workaround with its own failure mode; a paid tier is the fix. Record it as a risk |
| `supabase` | Egress allowance (around 5 GB/month), images served from the storage host | Sets the per-image ceiling — `rules-media.md` §4 |
| `cloudflare` | D1: 5M rows read/day, 100k rows written/day, 5 GB; over quota, queries fail until midnight UTC | A usage widget in the admin; content pages unaffected |
| `cloudflare` | R2: 10 GB, free egress | The egress arithmetic stops being about money — `rules-media.md` §4 |
| `cloudflare` | Workers Builds: 3,000 build minutes/month, one build at a time | Publishing is a rebuild; a busy editor queues builds |
| `cloudflare` | Access: 50 users | Fine for staff; never for the public |
| either, if `notifications` | Resend: 3,000 emails/month, 100/day, one webhook endpoint | Caps reminder volume — `rules-notifications.md` |
