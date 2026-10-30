---
name: supabase-app-hardening
description: Use when building or reviewing a multi-tenant Supabase app, creating tables and RLS policies, writing SECURITY DEFINER functions, wiring permission checks, or running bulk RPCs and migrations against a live database. Also use when a write reports success but changes nothing, a query dies at ~8 seconds, a paginated read skips or duplicates rows, a permission looks granted but has no effect, or a user sees another tenant's data. Triggers / gatilhos: RLS, row level security, policy, política, SECURITY DEFINER, statement_timeout, paginação, .range(), multi-tenant, permissão, migration em produção, escrita silenciosa.
---

# Supabase App Hardening

## Overview

Every rule here was learned from a production incident in a real multi-tenant
financial app, not from documentation. They share one property: **the failure is
silent.** No exception, no error in the console, no red in the logs, just wrong
data, missing rows, or a user quietly seeing what they should not.

**Core principle: in Postgres + Supabase, the dangerous failures do not raise.
Prove every security and scale decision against the database; never infer it from
the code.**

## When to Use

- Creating any table that holds tenant data
- Writing or editing an RLS policy
- Writing a `SECURITY DEFINER` function or any RPC
- Adding a permission key, role, or preset
- Writing an RPC that touches more than a few thousand rows
- Reading a collection that might exceed 1000 rows
- Reviewing someone else's migration before it reaches production

**Not for:** organizing multiple apps into schemas inside one project (that is a
separate concern), local-only prototypes with no tenants, or single-user tools.

## Day-Zero Setup

Create these foundations before the first feature table. Retrofitting them costs
far more than starting with them.

### 1. Tenant and membership

```sql
create table public.companies (
  id uuid primary key default gen_random_uuid(),
  razao_social text not null,
  created_at timestamptz not null default now()
);

create table public.company_users (
  id uuid primary key default gen_random_uuid(),
  company_id uuid not null references public.companies(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  is_admin boolean not null default false,
  is_active boolean not null default true,
  unique (company_id, user_id)
);
```

### 2. The four helper functions every policy will call

```sql
create or replace function public.belongs_to_company(_user_id uuid, _company_id uuid)
returns boolean language sql stable security definer set search_path = public as $$
  select exists (select 1 from public.company_users
                  where user_id = _user_id and company_id = _company_id
                    and is_active);
$$;

create or replace function public.is_company_admin(_user_id uuid, _company_id uuid)
returns boolean language sql stable security definer set search_path = public as $$
  select exists (select 1 from public.company_users
                  where user_id = _user_id and company_id = _company_id
                    and is_active and is_admin);
$$;
```

Add `has_role(user, role)` and `has_permission(user, company, key)` in the same
shape. **`has_permission` must require an active membership**, otherwise a
deactivated user keeps every permission they had.

### 3. The permission preset lives in SQL

```sql
create or replace function public.get_role_preset_permissions(_role text)
returns text[] language sql immutable as $$
  select case _role
    when 'admin'   then array['settings.edit','users.manage', ...]
    when 'analyst' then array['reports.view', ...]
    else array[]::text[]
  end;
$$;
```

**A TypeScript array of the same presets is a UI preview, never the source of
truth.** `has_permission` and every RLS policy resolve through the SQL function.
Granting a permission only in TypeScript produces a permission that exists on
screen and does nothing in the database.

### 4. The policy template: all four verbs, every table

```sql
alter table public.<t> enable row level security;

create policy <t>_select on public.<t> for select
  using (public.belongs_to_company((select auth.uid()), company_id));

create policy <t>_insert on public.<t> for insert
  with check (public.is_company_admin((select auth.uid()), company_id)
              or public.has_permission((select auth.uid()), company_id, '<key>'));

create policy <t>_update on public.<t> for update
  using      (public.is_company_admin((select auth.uid()), company_id)
              or public.has_permission((select auth.uid()), company_id, '<key>'))
  with check (public.is_company_admin((select auth.uid()), company_id)
              or public.has_permission((select auth.uid()), company_id, '<key>'));

create policy <t>_delete on public.<t> for delete
  using (public.is_company_admin((select auth.uid()), company_id));
```

## The Non-Negotiable Rules

### 1. Four policies per table. Always.

A table with RLS enabled and **no UPDATE policy** does not reject updates, it
matches **zero rows**, returns `error = null`, and the client shows a success
toast. The user believes the record was saved. It was not.

The same holds for DELETE and INSERT. Enabling RLS without a policy for a verb
silently disables that verb.

```sql
-- verify: expect four rows, one per command (r=select a=insert w=update d=delete)
select polname, polcmd from pg_policy
 where polrelid = 'public.<t>'::regclass order by polcmd;
```

**Defensive client code**, because the failure is silent, treat "0 rows" as an
error at the call site:

```ts
const { data, error } = await supabase
  .from('t').update(patch).eq('id', id).select().maybeSingle();
if (error) throw error;
if (!data) throw new Error(
  'Nothing was updated. The record may have changed, or you may lack permission to edit it.'
);
```

Word that message to name **both** causes. Blaming concurrency alone sends the
user into an infinite retry loop against a permission problem.

### 2. `SECURITY DEFINER` bypasses RLS: so it needs its own gate

A `SECURITY DEFINER` function runs as its owner and **ignores every policy on the
tables it touches**. If it accepts a tenant id as a parameter and is granted to
`authenticated`, any logged-in user can pass any tenant id.

```sql
create or replace function public.fn_something(p_company uuid, ...)
returns ... language plpgsql security definer set search_path to 'public' as $fn$
begin
  if not (public.is_super_admin(auth.uid())
          or public.is_company_admin(auth.uid(), p_company)
          or (public.has_role(auth.uid(), 'company_admin')
              and public.belongs_to_company(auth.uid(), p_company))
          or public.has_permission(auth.uid(), p_company, '<key>')) then
    raise exception 'Sem permissão' using errcode = '42501';
  end if;
  ...
end;
$fn$;
```

Where the function can derive the tenant from an existing row, **derive it**
instead of trusting the parameter, that removes the attack surface entirely.

### 3. `REVOKE` before `GRANT`, on every function

Postgres grants `EXECUTE` on new functions to `PUBLIC` by default. `GRANT EXECUTE
TO authenticated` does not remove that.

```sql
revoke all on function public.fn_something(uuid, int) from public, anon;
grant execute on function public.fn_something(uuid, int) to authenticated;
```

Order matters: revoke first, then grant. Verify:

```sql
select grantee, privilege_type from information_schema.routine_privileges
 where routine_name = 'fn_something';
-- expect: authenticated, postgres, service_role, never PUBLIC or anon
```

### 4. Admin is often two things at once

Mature codebases accumulate two representations of "admin": an application role
(`has_role(user,'company_admin')`) and a flag on the membership row
(`company_users.is_admin`). A gate that checks only one locks out half the admins.
Honor both, as in the template above.

### 5. `authenticated` dies at 8 seconds

The `authenticated` role carries a short `statement_timeout`. Any RPC that scans
or writes a large table must raise it **inside** the function:

```sql
set local statement_timeout to '120s';
```

`SET LOCAL` scopes it to the transaction. Also set it on the function itself
(`alter function ... set statement_timeout to '120s'`) so it survives replacement.

### 6. `.range()` without a unique order silently corrupts pagination

PostgREST's `.range()` maps to `LIMIT/OFFSET`. Without a **unique** `ORDER BY`,
Postgres may return rows in a different order per page, so a paginated read
**skips some rows and duplicates others**, with no error.

```ts
// ✅ always
.order('id').range(from, to)
```

A timestamp is not a tiebreaker: bulk imports write thousands of rows with an
identical `created_at`. Order by the primary key, or add it as the final key.

### 7. Read big collections page by page: sequentially

PostgREST caps a response at 1000 rows. Anything larger needs a loop. **Do not
parallelize that loop**: firing the pages concurrently saturates the connection
pool and trips the statement timeout. A sequential loop that works beats a
parallel one that times out.

The real fix for a slow aggregate is not a faster client loop, it is an
aggregating RPC that returns the summary instead of the rows.

### 8. Wrap `auth.uid()` in a subquery inside policies

```sql
-- ❌ re-evaluated per row
using (public.belongs_to_company(auth.uid(), company_id))

-- ✅ evaluated once (InitPlan)
using (public.belongs_to_company((select auth.uid()), company_id))
```

On a table with tens of thousands of rows this is the difference between a
`count(*)` taking seconds and taking milliseconds. Preserve this form whenever you
edit an existing policy, silently dropping it is a pure performance regression
that no test catches.

### 9. Read the live definition before editing a function

Repository migration history is frequently incomplete: functions get replaced by
later migrations, hotfixes, or dashboard edits. Copying a function body from an
old migration file and re-applying it **reverts every change made since**.

```sql
select pg_get_functiondef('public.fn_something'::regproc);
```

Edit what production actually runs, not what the repo remembers.

### 10. Migrations against a live database are additive, and proven

- New column: nullable, no backfill unless the backfill is the point. A backfill
  that "completes" the data can silently exclude rows from every screen that does
  not yet filter on the new column.
- Prove the invariant before and after, with the same query. Freeze the output of
  the key aggregates first; compare to the cent afterwards.
- Never apply a migration whose rollback you cannot state in one sentence.

## Quick Reference

| Symptom | Cause | Fix |
|---|---|---|
| Save succeeds, nothing changes | No UPDATE policy, or policy excludes this user | Four policies; check `pg_policy`; throw on 0 rows |
| Permission granted, no effect | Granted only in the TS preset | Add it to the SQL preset function |
| Query dies at ~8s | `authenticated` statement_timeout | `set local statement_timeout to '120s'` |
| Export has missing/duplicate rows | `.range()` without unique order | `.order('id')` |
| Bulk read times out | Parallel pagination | Sequential loop, or an aggregating RPC |
| `count(*)` takes seconds | `auth.uid()` re-evaluated per row | `(select auth.uid())` |
| User sees another tenant | `SECURITY DEFINER` without a gate | Gate on the tenant parameter, or derive it |
| Anon can call an RPC | Default `PUBLIC` execute grant | `revoke all ... from public, anon` |
| Deactivated user keeps access | `has_permission` ignores `is_active` | Require active membership |

## Verification Recipes

Never conclude from reading code. Impersonate and observe, inside a transaction
you roll back:

```sql
begin;
set local role authenticated;
set local request.jwt.claims = '{"sub":"<USER_UUID>","role":"authenticated"}';

-- should return rows for the user's own tenant
select count(*) from public.<t>;

-- should affect 1 row for an authorized user, 0 for an unauthorized one
update public.<t> set name = name where id = '<row>';

-- should raise 42501 for a foreign tenant
select public.fn_something('<other_company_uuid>');
rollback;
```

Run the negative case too. A test that only proves the allowed path passes
against a policy that allows everyone.

## Common Mistakes

| Mistake | Reality |
|---|---|
| "RLS is on, so the table is protected" | RLS on with no policy for a verb silently disables that verb |
| "The function is `SECURITY DEFINER`, so it's privileged and fine" | That is exactly why it needs its own gate, it skips RLS |
| "I granted it to `authenticated`, so `anon` can't call it" | `PUBLIC` keeps the default grant until you revoke it |
| "The permission is in the role preset" | Only if the preset the database reads is the SQL one |
| "It worked in my test" | Test the denial, not just the grant |
| "I'll copy the function from the migration file" | The repo may be behind production; read `pg_get_functiondef` |
| "Parallel pagination will be faster" | It saturates the pool and trips the 8s timeout |
| "I'll backfill the column so the data is complete" | Backfill can hide rows from every screen not yet year/scope-aware |

## Red Flags: Stop and Verify

- You wrote a policy and did not count four of them
- You wrote `security definer` and no `raise exception ... 42501`
- You wrote `grant execute` with no `revoke` above it
- You added a permission key and only touched TypeScript
- You wrote `.range()` and the query has no `.order('id')`
- You edited a policy and the `(select auth.uid())` form disappeared
- You are about to apply a migration and cannot state its rollback
