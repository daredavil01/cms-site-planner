# Schema and policies

How to turn the interview's content types into tables, buckets and policies. This produces the
*plan* for a migration, not the migration itself.

---

## 1. Derive the table list from what the site publishes

One table per content type the interview named. Two extra rules:

- **Config singletons** — site settings, contact details, homepage hero — go in one or two
  key-value or single-row tables, not one table per section.
- **A child that is only ever shown with its parent** is a child table with a FK and a
  `sort_order`, not an array column. Arrays cannot be reordered by an admin without rewriting the
  whole row.

Give every content table: `id`, `slug` where it has a public URL, `sort_order`, `published`
boolean, `created_at`, `updated_at`. If the `i18n` module is on, every translatable column is
JSONB (see `rules-i18n.md`).

## 2. Sort tables into access classes, then emit policies from a loop

**Rule.** Group tables by access class and emit each class's policies from a single loop in the
migration.

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

## 4. Buckets

One public bucket for anything a public page renders. A **separate private bucket** for anything
that must not be, kept out of the URL helper's type union (`rules-media.md` §2). Put the size
ceiling on the bucket itself as well as in the upload path — a client-side limit is a convenience,
not a control.

## 5. Derive state rather than storing it

**Rule.** If a value can be computed from other columns, compute it. A stored duplicate is a
second source of truth that will disagree with the first.

The exception is a value that is expensive and genuinely static once set — record *why* it is
stored, next to the column.

## 6. Avoid reserved words in column names

The reserved word is cheaper to avoid than to quote. `order`, `end`, `user`, `default` and
`references` all need quoting forever, in every query, by everyone.

## 7. Migration file discipline

- Numbered, sequential, applied once. The first file is **not** re-runnable — its `create policy`
  statements throw on a second run. Never edit it to make it re-runnable; write a new file.
- UTF-8 **without a BOM**, always, and especially when the file carries non-Latin seed data.
- Everything about drift, `migration list`, and `migration repair` is in `rules-core.md` §12, and
  the generated `CLAUDE.md` must carry it.
