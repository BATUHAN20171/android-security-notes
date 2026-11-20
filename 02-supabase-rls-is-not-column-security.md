# 02 — Supabase: one table without RLS, and RLS that protects rows but not columns

*Found in the backend of one of my own apps (a small private mobile/web app on Supabase) during a self-audit, fixed, and verified with before/after tests on a local Postgres. The app is private, so the examples below use generic table and column names (`daily_prompts`, `groups`, `profiles`, `group_id`) and simplified SQL.*

## The situation

The app lets a small, closed group of people share things. The data model is the usual multi-tenant one:

- `groups`: one row per group, with a `join_code` that new members type in,
- `profiles`: one row per user, with a `group_id` column saying which group they belong to,
- a dozen feature tables (`events`, `photos`, `lists`, ...), all with a `group_id`,
- `daily_prompts`: a fixed list of ~100 "question of the day" texts. Every day the app picks one and also sends it as the **body of a push notification**.

Every feature table is protected the same way: Row Level Security (RLS) on, and policies like

```sql
create policy "members read" on public.events
  for select to authenticated
  using (group_id = public.my_group_id());
```

where `my_group_id()` is a `security definer` helper that returns `profiles.group_id` for `auth.uid()`. Joining a group only happens through an RPC that checks the code and the member limit:

```sql
create function public.join_group(code text) returns void
language plpgsql security definer set search_path = public as $$
begin
  -- checks: logged in, not already in a group, code exists, group not full
  update public.profiles set group_id = target where id = auth.uid();
end $$;
```

And the profile policy looked harmless:

```sql
create policy "update myself" on public.profiles
  for update to authenticated
  using (id = auth.uid())
  with check (id = auth.uid());
```

## Problem 1: the one table without RLS

`daily_prompts` was "just static content", so its migration had a `create table` and an `insert`, but **no `enable row level security`** and no `revoke`.

On Supabase, tables you create with SQL in the `public` schema normally receive table privileges for the `anon` and `authenticated` roles through default privileges (unless the project's Data API defaults were changed). The publishable/anon key ships inside every client bundle, by design. RLS is what makes that combination safe. With RLS off, the grants are the only gate, and they say yes:

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| `anon` (no login) | yes | yes | yes | yes |
| `authenticated` | yes | yes | yes | yes |

So anyone who opens the site could, without an account, rewrite every prompt through the REST API. That is worse than it sounds: the server itself sends the prompt text as a push notification, so a stranger could make **the app's own notifications** say "Your account is locked, verify here: ...". They could also add rows until the free-tier database is full. (CWE-862, *Missing Authorization*.)

## Problem 2: RLS protects rows, not columns

The `update myself` policy checks *which row* you update. It says nothing about *which columns*. And the table-level `UPDATE` grant covers every column, including `group_id`.

That column is the key to everything: every policy in the schema trusts `my_group_id()`, which reads it. So a logged-in user could skip `join_group()` entirely (code check, member limit, "already in a group" check) by updating their own row:

```http
PATCH /rest/v1/profiles?id=eq.<my id>
{ "group_id": "<another group's id>" }
```

After that, every RLS policy honestly evaluates to "member" for that group. The attacker needs an account and the target group's UUID (not public, but UUIDs leak: former members, shared storage URLs that contain the group folder, screenshots). The same pattern existed on `groups`: members could update `join_code` and other columns the app never edits. (CWE-915, *Improperly Controlled Modification of Object Attributes*, a.k.a. mass assignment.)

## The fix

One migration, three parts.

**1. Turn RLS on for the forgotten table, allow only reading, and take the write grants away.**

```sql
alter table public.daily_prompts enable row level security;

create policy "logged-in users read" on public.daily_prompts
  for select to authenticated using (true);

revoke all on table public.daily_prompts from anon;
revoke insert, update, delete, truncate, references, trigger
  on table public.daily_prompts from authenticated;
revoke all on sequence public.daily_prompts_id_seq from anon, authenticated;
```

RLS alone would already block writes (no write policies), but removing the grants is cheap defense in depth, and `TRUNCATE` is not covered by RLS at all.

**2. Make the authorization column unwritable from the client.** First list what the client really writes. I searched the frontend for every `.update(`, `.upsert(`, `.insert(` and `.rpc(` touching these tables: it updates only `display_name` and `bio` on `profiles`, and `name` and `cover_image` on `groups`. Everything else (`group_id`, `last_seen_at`, status fields) is written by `security definer` functions or triggers, which run as the table owner and are not affected by these grants.

```sql
revoke insert, update, delete, truncate, references, trigger
  on table public.profiles from anon, authenticated;
grant update (display_name, bio) on table public.profiles to authenticated;

revoke insert, update, delete, truncate, references, trigger
  on table public.groups from anon, authenticated;
grant update (name, cover_image) on table public.groups to authenticated;
```

**The order matters.** In Postgres, a table-level `UPDATE` grant covers all columns, and a column-level `revoke update (group_id)` does nothing while it exists. You have to revoke at table level first, then grant the columns you want.

**3. Don't break what still needs access.** I left `SELECT` grants alone on purpose. A scheduled keep-alive job reads `groups` with the anon key; RLS returns zero rows to `anon`, but revoking `SELECT` would turn that into a permission error and fail the job.

## How I verified it

I didn't want "the migration looks right". I wanted the same attacks to succeed before and fail after, and the real app flows to keep working. I also didn't want to touch the live project. My machine has no Docker or Postgres, so I used [PGlite](https://github.com/electric-sql/pglite) (Postgres compiled to WebAssembly, runs inside Node) and recreated the parts of Supabase the policies depend on:

```sql
create role anon nologin;  create role authenticated nologin;
alter default privileges in schema public grant all on tables    to anon, authenticated;
alter default privileges in schema public grant all on sequences to anon, authenticated;

create function auth.uid() returns uuid language sql stable as $$
  select (nullif(current_setting('request.jwt.claims', true), '')::jsonb ->> 'sub')::uuid
$$;
```

Then the test script loads all real migrations into two fresh databases, *before* (without the new migration) and *after* (with it), seeds users and groups **through the app's own RPCs**, and runs each case the way PostgREST does: inside a transaction, `set local role ...`, set the JWT claims GUC, run the statement, roll back.

```js
await pg.exec('begin');
await pg.exec('set local role authenticated');
await pg.query(`select set_config('request.jwt.claims', $1, true)`,
               [JSON.stringify({ sub: attackerId, role: 'authenticated' })]);
await pg.query(`update public.profiles set group_id = $1 where id = $2`, [otherGroup, attackerId]);
// ...record result, then
await pg.exec('rollback');
```

Results (22 cases in total; a selection):

| Case | Before | After |
|---|---|---|
| anon: insert into `daily_prompts` | 1 row inserted | permission denied |
| anon: update all prompt texts | 100 rows changed | permission denied |
| anon: delete a prompt | 1 row deleted | permission denied |
| member: reads today's prompt via RPC + join | works | works |
| user: set own `group_id` to another group | 1 row changed, sees the other group, 3 members visible | permission denied |
| user: update own `display_name` / `bio` | works | works |
| member: change `join_code` | 1 row changed | permission denied |
| member: update group `name` / `cover_image` | works | works |
| `create_group()` + `join_group(code)` | works | works |
| `join_group()` into a full group | rejected | rejected |
| `last_seen` RPC, status trigger | works | works |
| anon: read `groups` (keep-alive) | 0 rows, no error | 0 rows, no error |

Two notes on reading results like these. First, an update filtered out by an RLS `using` clause does not fail; it changes **0 rows**. Only a missing grant gives "permission denied". Your test has to check the row count, not just "no error". Second, the local check doesn't prove the live state. For that I use read-only catalog queries in the SQL editor, before and after deploying:

```sql
select relrowsecurity from pg_class where oid = 'public.daily_prompts'::regclass;  -- expect true

select r.rol,
  has_table_privilege(r.rol, 'public.daily_prompts', 'INSERT') as ins,
  has_table_privilege(r.rol, 'public.daily_prompts', 'UPDATE') as upd
from (values ('anon'), ('authenticated')) r(rol);                                   -- expect all false

select has_column_privilege('authenticated', 'public.profiles', 'group_id', 'UPDATE'); -- expect false
```

After deploying, every catalog check on the live project matched the expected value, and an anonymous `GET /rest/v1/daily_prompts` with the public key now gets `401` with `42501 permission denied for table daily_prompts`.

## Lessons

1. **"Static content" tables are still tables.** Any table in an exposed schema needs RLS, even if the app only reads it. Put `enable row level security` and `revoke all ... from anon` in your table template so you can't forget them.
2. **RLS answers "which rows", grants answer "which columns".** If a column decides authorization (`group_id`, `org_id`, `role`, `is_admin`), the user must not be able to write it on their own row. Use column-level grants and change it only through a `security definer` function that does the checks.
3. **Revoke at table level, then grant columns.** A column-level revoke under a table-level grant is a no-op.
4. **Test the attack, not the migration.** Run the same malicious request before and after the fix, check row counts, and also test the real flows the fix could break.
5. Quick check you can run on your own Supabase project (read-only):

```sql
-- tables in public with RLS off
select c.relname from pg_class c join pg_namespace n on n.oid = c.relnamespace
where n.nspname = 'public' and c.relkind in ('r', 'p') and not c.relrowsecurity;

-- columns the authenticated role can update; look for anything that grants access
select table_name, column_name from information_schema.column_privileges
where grantee = 'authenticated' and privilege_type = 'UPDATE' and table_schema = 'public'
order by 1, 2;
```

Supabase's Security Advisor also flags tables with RLS disabled in `public`. It will not flag problem 2, though: column privileges are a design decision only you can review.
