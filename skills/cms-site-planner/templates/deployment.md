# deployment.md — template

> **To the skill.** Fill every `{{placeholder}}` you can from the interview; where nothing exists
> yet, write the row with its intended name and "not created". Keep the section headings even when a
> section is short — later phases fill them. Delete this blockquote from the generated file.

---

# {{PROJECT_NAME}} — Deployment requirements

**Last updated:** {{DATE}} · **Cloudflare account:** {{account name}} — check `npx wrangler whoami`
shows it before every deploy.

Everything needed to put the site into production and keep the preview environment matching it.
**Every phase that adds a migration, a binding, a variable, a secret or an outside service updates
this file in the same commit.** It is the only place those requirements are collected; a value
known only to the session that added it is a value production will be missing.

Running the live site afterwards — backups, restore, uptime, staff changes, incidents — belongs in
an operations runbook, written in the launch-readiness phase.

## 1. Where things stand

| Item | Production | Preview |
|---|---|---|
{{One row per resource: Cloudflare account, domain and DNS, database, storage buckets, each Worker,
staff sign-in, bot check, email sending, repository and CI. Each cell: the name, and "exists",
"not created" or "deployed on <date> at <URL>".}}

## 2. Blockers before the first deploy

{{Numbered. Anything in the setup checklist that is not verified, any `[TBD]` that the release
check will refuse, any resource not yet created.}}

## 3. Database

### 3.1 Migrations

| File | Phase | What it does |
|---|---|---|
{{Filled as migrations are written. Note any migration that rebuilds a table, any that must run
before a seed file, and any that is not safe to re-run.}}

### 3.2 Seeds and answers

{{Which seed files run where: the structural seed, the client-answers file (rerunnable, guarded on
seed values), the production owners' accounts. Development-only seeds never run against preview or
production.}}

### 3.3 Applying to production and to preview

{{The exact commands, in order, from the folder they run in. On `supabase`: check `supabase
migration list` first. On `cloudflare`: `wrangler d1 migrations apply DB --remote` (and `--env
preview`), then the seed files that apply.}}

### 3.4 Rules for later changes

- Never edit an applied migration; write the next one.
- Dry-run a risky remote change against an export first.
- {{Stack-specific rules from the chosen rules file.}}

## 4. Variables, secrets and bindings

One table per Worker or app, and one for the build environment and one for CI if they need values.
**A secret's value is never written here** — only its name, where it is set, and how to generate it.

### 4.1 {{Worker or app name}}

| Name | Kind | Production | Preview | Local (`.dev.vars`) | Added in phase | Notes |
|---|---|---|---|---|---|---|
{{Kind is var, secret or binding. For values that must agree with another Worker's, say so in
Notes — the config test checks them (rules-core.md §18).}}

## 5. Outside services

| Service | Used for | Account owner | Set up? | Notes |
|---|---|---|---|---|
{{DNS, email sending and its SPF/DKIM/DMARC records, bot check, staff sign-in, analytics, search
console, uptime monitoring — whatever the plan uses.}}

## 6. Deploy order

{{Numbered steps for the first production deploy: account check, database, secrets (after the first
deploy where the platform requires it), each Worker in dependency order, DNS and custom domains,
then the checks in §7.}}

## 7. Checks after each deploy

1. `npx wrangler whoami` showed the right account **before** the deploy.
2. Read the variable list the deploy printed — not the config file.
3. `curl https://<domain>/api/health` → 200, all checks named.
4. The release check against the deployed origin passes (`--launch` for production: no `[TBD]`).
5. The staff area refuses a signed-out request and admits a signed-in one.
{{Stack- and module-specific checks — on `cloudflare`: the Access login redirect and the certs URL
answer as expected; scheduled triggers are listed; a test email arrives from the right sender.}}

## 8. Done so far

| Date | Environment | What was deployed | Commit |
|---|---|---|---|
| | | | |

## 9. Client answers needed before launch

The open questions in [`status.md`](status.md) whose answers set launch values. Launch waits for
these; development does not.
