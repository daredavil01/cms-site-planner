# Module `privacy` — personal data beyond a contact message

Load only when the interview selected `privacy`: the site stores personal data that would harm
someone if exposed — health, children, finances, legal matters, bookings tied to a name — or a
data-protection law applies to what it keeps. A contact form alone needs `forms`, not this.

Format: **Rule** · *Why* · *Evidence*.

---

## 1. Write the exposure rules down as a checklist, and review the build against it

**Rule.** The requirements carry a checklist of exposure rules — collection, storage, public site,
admin, transmission, logs, backups — and a launch-readiness phase reviews the finished build against
it item by item, recording where each rule is enforced and tested. Repeat the review before launch
and on a schedule after.

*Why:* tests check the rules someone thought to test. A review walks every rule, including the ones
that live in configuration rather than code.

*Evidence:* a review at the end of a fully tested build found five problems no test had caught:
request logs keeping private-page tokens and IP addresses (`rules-core.md` §19), an error log that
could quote input (`rules-core.md` §13), two deploy-config values that would have broken outgoing
messages (`rules-core.md` §18), and a privacy notice without a contact for data questions.

## 2. Personal data never travels where it can be copied

**Rule.** No personal data in URLs, query strings, redirects, page titles, browser storage,
analytics events, logs or error messages.

- A private page is reached by a **random token** (32 bytes), and only its hash is stored.
- Its title is neutral ("Your booking", not the person's name or the service).
- Staff searches by name or number are **POST** forms, never GET; list filters that are safe to
  share stay in the query string.
- Private pages send `Cache-Control: no-store`, `noindex` and `Referrer-Policy: no-referrer`.
- Browser storage holds preferences only — a language, a theme — and the privacy notice says so.

*Why:* each of those places is copied somewhere you do not control — browser history, a shared
screenshot, an analytics dashboard, a referrer header, a support ticket.

## 3. Personal fields are encrypted in the application, bound to their row

**Rule.** Encrypt each personal field with AES-GCM before it reaches the database, using the
**table, column and row id as additional authenticated data**. The key lives only in the platform's
secret store. Every encrypted column has a check constraint that refuses plain text.

```sql
name_enc TEXT NOT NULL CHECK (name_enc GLOB 'v1.*')   -- Postgres: CHECK (name_enc LIKE 'v1.%')
```

*Why:* a database export, a backup, or a stray admin query then exposes ciphertext only.

*Evidence:* the two details are what make it hold.

- **Binding to the row** means a ciphertext copied into another row or column fails to open. The
  admin shows "[cannot be decrypted]" instead of the wrong person's name. Row ids must exist before
  sealing, so generate them in the application (UUIDs), not in the database.
- **The plain-text guard** turns a forgotten `seal()` into an insert error at development time
  rather than a readable column in production.

Prefix the format (`v1.`) so the key can be rotated later by adding a key id; write down that there
is no rotation yet, if there is none.

## 4. The public side cannot reach personal tables at all

**Rule.** The public read path is structurally unable to read personal tables, and a test proves it
with personal rows present. On `supabase`: no anon `select` on those tables. On `cloudflare`: they
are absent from the export allow-list (`rules-cloudflare.md` §1).

*Evidence:* a test that runs against empty personal tables passes whether or not the boundary
exists (`rules-core.md` §4). Insert personal rows first, then assert none reach the public output.

Public endpoints return schedules, labels and counts — never another person's record.

## 5. Staff see the least, and every view is logged

**Rule.** The role table decides which staff see personal data at all. Lists show what is needed to
pick a record and hide the sensitive part (topic, message) until it is opened. Every opening and
every export writes an audit row. Exports carry totals or metadata only — no bulk export of
personal data.

*Why:* "who looked at this record" is the first question in any complaint or breach, and only a log
written at the time can answer it.

## 6. Retention is a scheduled job with tests, and its deletes must not be blocked

**Rule.** Every personal table has a retention period and a purge date column. A nightly job deletes
what is past it, plus expired sessions and rate-limit rows. Triggers remove dependent rows (locks,
sessions, notes) with their parent and anonymise events kept for reporting. Tests cover each table.

*Evidence:* two interactions that would have stopped retention silently.

- **A foreign key `ON DELETE SET NULL` meets a check constraint.** A rule such as "no parent means
  this flag is 0" fails when retention deletes the parent and the key goes null — and the failure
  rolls back the **whole** retention batch. State such rules on a column the delete leaves alone.
- **Reports that count personal records must store their figures before retention deletes the
  rows.** A test compares the figures counted live, after the nightly statistics run, and after the
  rows behind them are deleted; all three must be equal.

## 7. Messages and calendar files carry no personal detail

**Rule.** Alerts to staff contain the alert type and a link to the admin, nothing else. Messages to
the person contain only what they need — date, time, place, their private link. For sensitive
services, use one neutral name ("Consultation") everywhere it can leak: page titles, confirmation
screens, messages, calendar files, analytics.

*Why:* staff email and phone notifications are read on lock screens and shared devices; a calendar
entry syncs to wherever the person's calendar goes.

## 8. Consent is recorded with the notice it was given against

**Rule.** Consent is an unticked box tied to a **versioned** privacy notice; store the version, the
language and the time with the record. When the notice changes, publish a new version — never edit
the old one.

*Evidence:* when the notice version is derived from published content, the seed's first version and
the derivation must agree; a test publishes a fresh seed and expects no new version.

## 9. Nothing on the page can read the form

**Rule.** No third-party script that can read form fields — no chat widgets, ad pixels or session
recorders. Analytics is cookieless and never receives form data or topics, and is absent from
private pages. Embedded maps load only after a tap.

## 10. Development and test data are made up

**Rule.** Seeds for development, test fixtures and documentation screenshots use invented people
only, generated in memory or from a dev-only seed file. Never copy preview or production data down.

*Why:* a screenshot in a user guide is published to every reader of the guide.
