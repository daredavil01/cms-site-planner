# Schema and access

How to turn the interview's content types into tables, buckets and access rules. This produces the
*plan* for a migration, not the migration itself. Where the two stacks differ, both are given; use
the one the interview chose.

---

## 1. Derive the table list from what the site publishes

One table per content type the interview named. Two extra rules:

- **Config singletons** — site settings, contact details, homepage hero — go in one or two
  key-value or single-row tables, not one table per section.
- **A child that is only ever shown with its parent** is a child table with a FK and a
  `sort_order`, not an array column. Arrays cannot be reordered by an admin without rewriting the
  whole row.

Give every content table: `id`, `slug` where it has a public URL, `sort_order`, `published`
boolean (or a status), `created_at`, `updated_at`. If the `i18n` module is on, translatable text is
JSONB on `supabase` and a translation table on `cloudflare` (see `rules-i18n.md` §3).

**Every status and type column gets a `CHECK` listing its values.** The list is the documentation,
and a typo in application code becomes an insert error instead of a row nothing will ever match.

## 2. Sort tables into access classes, then generate access from the class

**Rule.** Group tables by access class and generate each class's access from one list — on
`supabase`, a single policy loop in the migration; on `cloudflare`, the export allow-list and the
route table.

*Why:* hand-writing a policy per table is where the inconsistent one hides. Thirty-eight
near-identical policies invite a typo nobody will read closely enough to catch.

*Evidence:* a four-class schema — config singletons, published content, parent-gated children,
staff-only — plus a handful of exceptions called out individually. **The exceptions are visible
precisely because the rule is a loop.**

| Class | Anon `select` | Staff |
|---|---|---|
| Config singleton | `using (true)` | full |
| Published content | `using (published = true)` | full |
| Parent-gated child | `using (exists (… parent published …))` | full |
| Staff-only | none | full |
| Public-write surface | **none** — `insert … with check (true)` only | full |

```sql
do $$
declare t text;
begin
  foreach t in array array['programs', 'posts', 'team_members', 'partners'] loop
    execute format($f$
      create policy %1$I_public_read on public.%1$I
        for select to anon, authenticated
        using (published = true);
      create policy %1$I_staff_all on public.%1$I
        for all to authenticated
        using (public.is_staff()) with check (public.is_staff());
    $f$, t);
  end loop;
end $$;
```

**On `cloudflare` the same classes drive the export allow-list and the Worker's guards.** There
are no policies; the class decides whether a table is copied into the public build
(`rules-cloudflare.md` §1) and which routes may touch it:

| Class | In the export allow-list | Public routes | Admin |
|---|---|---|---|
| Config singleton | yes | — | by role |
| Published content | yes, published rows | — | by role |
| Parent-gated child | yes, when the parent is published | — | by role |
| Staff-only | **no** | none | by role |
| Public-write surface | **no** | insert through one endpoint (`rules-forms.md`) | by role |
| Live operational data (availability, schedules) | **no** | read-only endpoints returning no personal data | by role |

**Public-write surfaces get insert-only anon policies.** Contact, volunteer, registration and
newsletter tables accept `insert … with check (true)` and grant anon **no `select` at all** —
they hold names and phone numbers of people who contacted the organisation. This is the right
trade, and it is what makes the RPC in §3 necessary.

## 3. A `security definer` RPC, when a public page needs a fact from a private table

**Rule.** Reach for this before reaching for a credential that bypasses RLS.

*Why:* a function exposes exactly one derived fact without making a single row reachable. A bypass
credential exposes everything and relies on the calling code to be careful forever.

*Evidence:* a homepage needed counts of form submissions from tables that grant anon `insert` and
nothing else. `select count(*)` from an anonymous session returns `0` — **not because the tables
are empty, but because RLS filters every row.** A counter built on that reads zero forever and
looks perfectly healthy doing it.

```sql
create or replace function public.public_counters()
returns table (submissions bigint, registrations bigint)
language sql
stable
security definer
-- Pinning search_path is not optional on a security-definer function: without
-- it a caller can prepend a schema and have their own table resolved instead,
-- running it with the owner's rights.
set search_path = public, pg_temp
as $$
  select
    (select count(*) from public.contact_submissions
       where status is distinct from 'declined'),
    (select count(*) from public.event_registrations);
$$;

-- Revoke first: a new function is executable by PUBLIC by default, and PUBLIC
-- is wider than the two roles that should have it.
revoke all on function public.public_counters() from public;
grant execute on function public.public_counters() to anon, authenticated;
```

Three things there are the actual lesson, and all three are easy to omit: **pin `search_path`**,
**`revoke` before you `grant`**, and **return aggregates, never rows**.

On `cloudflare` the same rule is a Worker endpoint, or a nightly statistics table, that returns the
aggregate and nothing else.

## 4. Buckets

One public bucket for anything a public page renders. A **separate private bucket** for anything
that must not be, kept out of the URL helper's type union (`rules-media.md` §2). Put the size
ceiling on the bucket itself as well as in the upload path — a client-side limit is a convenience,
not a control.

On `cloudflare`: one media bucket that the build copies images from, documents streamed through a
Worker route (`rules-media.md` §7) so no bucket needs to be public, and a **private** bucket for
encrypted backups (`rules-cloudflare.md` §10).

## 5. Derive state rather than storing it

**Rule.** If a value can be computed from other columns, compute it. A stored duplicate is a
second source of truth that will disagree with the first.

The exception is a value that is expensive and genuinely static once set — record *why* it is
stored, next to the column.

**Stored statistics store totals and counts, never averages.** An average per day weights a quiet
day like a busy one; store the sum and the count and divide over the chosen range. Keep "first
time" columns separate from "last changed" ones — a `handled_at` that moves with every status change
cannot give a response time; a `replied_at` set once (`coalesce(replied_at, …)`) can.

## 6. Avoid reserved words in column names

The reserved word is cheaper to avoid than to quote. `order`, `end`, `user`, `default` and
`references` all need quoting forever, in every query, by everyone.

## 7. Migration file discipline

- Numbered, sequential, applied once. Never edit an applied file to make it re-runnable; write a
  new file (`rules-core.md` §12).
- UTF-8 **without a BOM**, LF line endings, always — especially when the file carries non-Latin or
  multi-line seed data.
- Stack mechanics — drift and repair on `supabase` (`rules-supabase.md` §6), table rebuilds, restore
  order and dry runs on `cloudflare` (`rules-cloudflare.md` §7) — go into the generated `CLAUDE.md`.

## 8. Drafts and published copies

**Rule.** Decide per content type whether an edit is live on save or waits for a publish step, and
make the reader choose the right copy in one place.

- **Live on save** (`supabase` default): one row with a `published` flag. Editing a published row
  changes the public page at once.
- **Draft, then publish** (`cloudflare` default, and any type needing review): the draft and the
  published copy are separate rows — for translation tables, the state is part of the primary key
  (`… PRIMARY KEY (id, locale, status)`). One helper picks the visible row for a mode (draft
  preview or public). Publishing deletes the published row a ready draft replaces and flips the
  draft, in **one atomic write**.

*Evidence:* never join a translation table without the helper — in draft mode the join returns the
draft **and** the published copy, and the page shows both.

Operational settings that staff change during the day — schedules, closures, availability rules —
are usually live, not drafted. Decide it in the plan; a publish step on a closure means a closed day
stays bookable until someone remembers to publish.

## 9. Constraints that pass when you expect them to fail

**Rule.** Write each `CHECK` so that its failure case evaluates to `false`, not `NULL`.

*Evidence:* a `CHECK` that evaluates to `NULL` **passes** (SQL treats unknown as satisfied).
`CHECK (date(x) = x)` accepted `2026-13-01`, because `date()` returned `NULL`; `CHECK (date(x) IS x)`
refuses it (`IS NOT DISTINCT FROM` on Postgres).

**`ON DELETE SET NULL` meets `CHECK`.** A rule like "no parent means this flag is 0" fails when the
parent is deleted and the key goes null — and blocks whatever deleted the parent, often a whole
retention batch. State such rules on a column the delete leaves alone.

**Rows that must never change are protected by triggers**, not by the absence of an edit button —
approved template versions, sent messages, audit rows. Cleaning them up in development then means
dropping and recreating the trigger in the same run.

## 10. Seeds carry structure, markers and nothing invented

**Rule.** The seed holds the client's real content where it exists, `[TBD]` for unconfirmed facts,
`NULL` for unconfirmed limits, and empty strings for optional settings (`rules-core.md` §14a).
Content that is not real stays unpublished. The client's answers arrive later in a separate,
rerunnable answers file.

*Evidence:* seeds and application code must agree on anything derived — a notice version, a slug, a
generated label. A test runs the seed, runs the derivation, and expects no change.

Development users and fixtures live in a separate dev-only seed with made-up people
(`rules-privacy.md` §10). Production gets only the owners' accounts, from their own file.

## 11. Feature switches, when features launch at different times

**Rule.** If the interview named features that launch later or can be paused, give them switches in
one table with an owner-only editor and a `depends_on` column. Turning a parent off turns its
dependants off; pages and menu links for a disabled feature are not built; endpoints check the switch
on every request (a short cache is fine) and answer "unavailable" with a contact route.

*Why:* a feature then launches by flipping a row, not by deploying, and a broken one can be stopped
the same way.

