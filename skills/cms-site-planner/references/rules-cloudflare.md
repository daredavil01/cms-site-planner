# Stack `cloudflare` — static Astro + D1 + R2 + Access

Load only when the stack is `cloudflare` (`stacks.md`). Evidence comes from one full build with a
test suite, deployed to a preview environment. Production cutover had not happened when this was
written; say so in the generated `status.md`.

Shape of the stack: Astro builds static pages from **published** content exported out of D1; a site
Worker serves the dynamic routes (`/api/*`, private pages, downloads, cron jobs); a separate admin
Worker sits behind Cloudflare Access. Publishing marks drafts published and triggers a rebuild.

Format: **Rule** · *Why* · *Evidence*.

---

## 1. The build reads an allow-list of published tables, through a sandbox

**Rule.** The public build never queries the live database. A build step copies the rows of an
**allow-listed** set of published content tables into an empty, migrated sandbox database, and every
reader the build uses runs against the sandbox. A test asserts the allow-list contains no table
holding personal data. This is `rules-core.md` §4 on this stack: there is no RLS, so the allow-list
*is* the public access boundary.

*Why:* a query helper that "only reads published rows" is a habit; a sandbox that does not contain
the other tables is a fact.

*Evidence:* the release test inserts personal rows into the source database, runs the export, and
asserts none reach the sandbox or `dist/`. A second test checks every allow-listed table exists, so
a rename cannot silently empty a page.

**Run the export as an Astro integration** (`astro:build:start`, `astro:server:setup`), so `astro
build` and `astro dev` always see current content. Consequences to write into the plan: the build
needs a migrated and seeded local D1, so CI migrates and seeds before building; and tests that read
`dist/` run after the build.

**Load generated content with `import.meta.glob`.** A static JSON import makes `astro check` fail
before the first export has produced the file; a glob does not.

**Every image the content uses is copied into the build**, and the export **stops the build** when
one is missing from R2. Templates look images up through one helper that returns nothing for an
unknown key, so render tests can pass any key.

## 2. The site Worker: handlers only, and its own headers

**Rule.** The Worker's main module exports `default` and nothing else; constants live in other
modules. A test asserts it. Responses the Worker builds itself set their own security headers
(`rules-core.md` §20).

*Evidence:* an exported constant beside the handlers passed every Node test and stopped `wrangler
dev` with "Incorrect type for map entry". Start the Worker once in every phase that touches
its entry file.

**A Worker-rendered page inside the Astro layout.** Build a shell page in Astro with a marker
element; the Worker fetches it with `env.ASSETS.fetch(new URL('/shell/', url))`, swaps the title,
`html lang` and the marker, and adds no inline script or style, so the shell's CSP still holds. Use
**function replacements** in `String.replace` — with a string replacement, a `$&` in a visitor's
text is expanded. A dist test checks the real shell keeps its marker.

## 3. A D1 batch is your only transaction

**Rule.** Anything that must happen together is one `db.batch([...])`. A failing statement rolls
back the whole batch; there are no interactive transactions. Close read-then-write windows **inside**
the batch, never with a read before it.

*Why:* the time between a read and the batch that acts on it is where two requests both see a slot
free.

*Evidence:* three patterns, all tested, that made it work —

- **Unique index + batch.** A booking's row, its slot locks and its first event are one batch; a
  unique index on the locks fails the whole batch for the loser (`rules-booking.md` §2).
- **Stale guard.** `SET start_at = CASE WHEN <still as read> THEN ? END` writes NULL into a NOT NULL
  column when the row changed since it was read, failing the batch, and the caller reports
  "changed".
- **Claim by insert.** `INSERT … SELECT … ON CONFLICT DO NOTHING RETURNING` returns only the rows this
  run inserted, so overlapping runs split the work (`rules-notifications.md` §4).

**`db.batch([])` throws.** A list view with nothing to audit built an empty batch and answered 500.
Skip the call when there is nothing to write.

**`db.batch` works for reads too** — one round trip for a day's rules, closures and locks.

## 4. D1 limits that bite, with their workarounds

**Rule.** Write these into the generated `CLAUDE.md`; each one cost a session to find.

| Limit | Workaround |
|---|---|
| Five terms in a compound `SELECT` (`UNION ALL` of six tables fails) | Split the query, or count with scalar subqueries: `SELECT (SELECT count(*) FROM a) AS a, …` |
| Long `LIKE`/`GLOB` patterns refused ("pattern too complex", about 45 characters) | Validate by round trip: `x = strftime('%Y-%m-%dT%H:%M:00.000Z', x)` |
| `CREATE TEMP TABLE` refused (`SQLITE_AUTH`) | An ordinary scratch table |
| Triggers cannot use CTEs and do not recurse | Keep trigger cascades one level deep; deeper needs app code |
| A primary key cannot change in place | Rebuild the table: create new, copy, drop, rename |
| A column computed from `EXISTS(...)` or a comparison has no type affinity, so `IN ('1')` never matches | `CAST(... AS INTEGER)` |
| `\|\|` with NULL gives NULL | `coalesce()` nullable columns when concatenating |
| `D1Database.exec` splits on newlines and breaks multi-line `CREATE TRIGGER` | Split SQL files with Wrangler's `unstable_splitSqlQuery` |
| `wrangler d1 execute --command` with several statements did nothing, with no error | Put them in a file and use `--file` |

**Over the daily quota, every query fails until midnight UTC.** Static pages stay up; forms, booking
and the admin stop. Plan an admin widget showing usage against the free-tier limits.

## 5. Every test gets a real local D1

**Rule.** Tests get a fresh in-memory D1 (and R2) from Wrangler itself:

```ts
const proxy = await getPlatformProxy<{ DB: D1Database; FILES: R2Bucket }>({
  configPath: './wrangler.jsonc',
  persist: false,
});
```

Load migrations and seed files with Vite `?raw` imports, apply them, and run the real code against
the result. Run the Hono admin in-process with `app.request(url, init, env)`.

*Why:* a mocked database tests the mock. This one starts in about half a second, enforces foreign
keys, does not recurse triggers, and rolls back a failing batch — the three behaviours the code
relies on.

*Evidence:*

- `Promise.all` of three `worker.fetch` calls is a real race against local D1: all three passed the
  free-slot check before any batch ran, and only the unique index let exactly one through.
- Do not import Miniflare directly; the copy in `node_modules` is a transitive alpha.
- `?raw` imports avoid `@types/node`, which clashes with the Worker types.
- Run Vitest through the workspace (`npm test -w shared`): from another folder the relative
  `configPath` misses and every test fails at once.
- Export tests that build two databases with every migration ran near the 5 s default timeout and
  crossed it when one more migration landed. Give that suite 20 s.
- Fixture rows whose `created_at` defaults to now join today's seed rows in date-range tests; use
  dates far from today.

## 6. The admin is a Worker behind Access, and it verifies the token itself

**Rule.** Cloudflare Access shows the login page; the admin Worker still verifies the
`Cf-Access-Jwt-Assertion` token on **every** request — signature against
`https://<team>.cloudflareaccess.com/cdn-cgi/access/certs`, audience, expiry — then looks the email
up in its own users table and refuses unknown or deactivated ones (`rules-core.md` §8).

*Why:* Access in front is a second lock, not the only one. A misconfigured Access policy, a route
outside the application, or a request that reaches the Worker another way would otherwise be
trusted.

*Evidence:* tests sign tokens with a WebCrypto RSA key and stub `fetch` for the certs URL only; bad
signature, wrong audience and expired tokens are refused. What no local test caught: a team domain
missing one letter, which served no keys. **After deploy, check that an unauthenticated request is
redirected to `https://<team>.cloudflareaccess.com/cdn-cgi/access/login/…` and that the certs URL
answers 200.**

**The local sign-in stand-in is guarded twice:** a `DEV_ACCESS_EMAIL` variable *and* a request host
of `localhost`. Cloudflare routes by host, so a deployed Worker never sees `localhost` even if the
variable leaks. Changing the email to `editor@example.test` shows another role's screens.

**Hono specifics.** `csrf()` throws an `HTTPException`, and a catch-all `onError` turned it into a
500: return `err.getResponse()` for those first. Form posts under `Referrer-Policy: no-referrer` send
`Origin: null` — see `rules-forms.md` §5. Every admin response is `no-store`, `noindex` and
unframeable, and a test checks it.

**Local state is shared.** Run the admin with `wrangler dev --persist-to ../site/.wrangler/state`
(and `d1 execute --persist-to`), or each Worker gets its own empty database. Migrations still run
from the site.

## 7. Migrations, seeds and restores on D1

**Rule.** `wrangler d1 migrations apply DB --local` (or `--remote`) applies `migrations/` in order.
Seeds run after every migration. A restore orders inserts by foreign key, not by creation order, and
is tested against a real `wrangler d1 execute`, not only in a batch.

*Evidence:*

- **Restore failed where its unit test passed.** Migrations that rebuild a table (create, copy,
  drop, rename) leave a parent table after its children in `sqlite_master`.
  `wrangler d1 execute --file` does not run a file as one transaction, so `PRAGMA
  defer_foreign_keys` alone did not help; the dump now orders inserts by each table's
  `REFERENCES`. The unit test had used `db.batch` — one transaction — and now runs statements one at
  a time, as the real restore does.
- **Resetting a local database:** drop every trigger first, then tables by `rowid DESC`, then
  migrate and seed. Dropping a table that another table's trigger names fails.
- **Dry-run a remote change on its export.** `wrangler d1 export <db> --remote --output x.sql`, split
  the dump, load it in memory (triggers come before their tables, so retry failed statements until
  none are left), run the change twice, and only then touch the remote.
- **Immutable rows are kept by triggers**, so a cleanup that must delete one drops the trigger,
  deletes and recreates the trigger in one `--file` run.

## 8. Wrangler configuration

**Rule.** `wrangler.jsonc` is commented: every var says what it is for, which other Worker must agree
with it, and the decision behind its value. A config test checks the values that must agree
(`rules-core.md` §18).

- **Preview and production are environments of one config** (`env.preview`). Remote bindings
  (`"remote": true` on `DB` and `FILES` under `env.preview`) let a build read the preview D1 and R2
  without exporting them; top-level bindings and tests stay local. Opening the connection took about
  35 seconds. npm sets `npm_lifecycle_event` to the script name, which picks the source without
  `VAR=value` prefixes that `cmd.exe` cannot run.
- **Custom domains need no `workers.dev` subdomain.** `routes: [{ pattern, custom_domain: true }]`
  with `workers_dev: false` and `preview_urls: false`; Wrangler creates the DNS record and
  certificate. A new account has no `workers.dev` subdomain, and a first `wrangler deploy` without
  custom domains fails until one is registered.
- **Set secrets after the first deploy** with `wrangler secret bulk --env <env>` reading JSON from
  stdin (`rules-core.md` §5).
- **The `send_email` binding takes a plain object** — `env.EMAIL.send({ from, to, subject, text })` —
  so Worker code runs in Node tests with a stub; under `wrangler dev` it writes each message to
  `.wrangler/tmp/email/`. It sends only from the zone's own domain (`rules-notifications.md` §2).

## 9. Two cron triggers per Worker, dispatched inside

**Rule.** One trigger every 15 minutes and one nightly. Every scheduled job — expiry, reminders,
digests, retention, statistics, backups — is dispatched from one of those two by checking the
scheduled time in code.

*Evidence:* the free plan allows **five cron triggers per account**, and the preview Worker's count.
A site and its preview Worker with two triggers each already use four, so a weekly backup runs
inside the nightly job on one weekday, and every later job reuses the existing two.

Test a job at any instant through the local endpoint:

```bash
curl "http://127.0.0.1:8787/cdn-cgi/local/scheduled?cron=*/15+*+*+*+*&time=<epoch ms>"
```

The job reads `controller.scheduledTime`, so an evening digest can be checked at noon. Without
`cron`, the endpoint runs the first trigger.

## 10. Backups are yours to build

**Rule.** A weekly D1 dump, encrypted with a key derived from the field-encryption key, written to a
**private** R2 bucket and copied off Cloudflare by a scheduled CI job; a restore tested before launch
and quarterly after.

*Why:* every part of the stack sits with one vendor. An off-platform copy reduces data-loss risk; it
does nothing for downtime, and the plan should say so.

*Evidence:* the first real restore found the foreign-key ordering bug in §7. A backup nobody has
restored is a hypothesis.

## Known blind spots on this stack

Carry these into the generated `CLAUDE.md` §7.

- **Published content is only as fresh as the last build.** A publish whose deploy hook failed, or
  that ran locally with no hook, leaves the site on old content with nothing visibly wrong. The
  publish job row records "waiting" or "failed"; the admin must show it.
- **The admin's draft preview and the public site are different renderers.** Check the published
  result on the built site, not only the preview.
- **A spent D1 quota stops every dynamic route until midnight UTC**, silently from the outside.

## Commands the generated `CLAUDE.md` lists

Each with what it needs; fill the `.dev.vars` keys with made-up local values.

| Command | What it is for |
|---|---|
| `npm run lint`, `npm run typecheck`, `npm run build`, `npm test` | The CI steps. The build exports from the local D1, so migrate and seed first; the dist tests read `dist/`, so build before testing |
| `npm run preview -w site` | Build, then the Worker and static assets under `wrangler dev` on port 8787; needs `site/.dev.vars` |
| `npm run db:migrate -w site`, then `npm run db:seed -w site` | `wrangler d1 migrations apply DB --local`, then the seed files; seed once per fresh database |
| `npm run types -w site` / `-w admin` | `wrangler types … --strict-vars=false` after changing a `wrangler.jsonc`; move `.dev.vars` aside first |
| `npm run dev -w admin` | The admin on port 8789 against the site's local D1 (`--persist-to`), with `DEV_ACCESS_EMAIL` in `admin/.dev.vars` |
| `curl "http://127.0.0.1:8787/cdn-cgi/local/scheduled?cron=…"` | Run a cron job locally (§9) |
| `npm run release-check -w site -- <origin>` | Release checks against a running preview, or production at launch (`rules-core.md` §21) |
| `npx wrangler whoami` | Before every deploy (`rules-core.md` §18) |
