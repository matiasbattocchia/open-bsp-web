---
title: 5 Supabase Patterns I Learned Building an Open-Source WhatsApp Platform
published: false
description: Real-world patterns for multi-tenant RLS, event-driven triggers, JSON merging, and CI/CD without paid branching.
tags: supabase, postgres, opensource, whatsapp
cover_image:
---

I've been building [OpenBSP](https://github.com/matiasbattocchia/open-bsp-api) — an open-source WhatsApp Business Platform — on Supabase since 2023. Back then, Supabase was promising but not what it has become today. Over these years I got to know its features in depth.

Here are 5 patterns I came up with while building a production multi-tenant messaging platform. Some are elegant, some are workarounds. All of them are running in production.

## 1. Multi-Tenant RLS with Dual Auth and Role Hierarchy

Every table in OpenBSP belongs to an organization. Users can be members of multiple organizations with different roles (owner, admin, member). And on top of that, we also support API key authentication for service-to-service access.

The typical Supabase approach is to use `auth.uid()` in RLS policies. But what if you also need to support API keys? And what about role levels?

We solved it with a single function that every RLS policy calls:

```sql
create function public.get_authorized_orgs(
  role public.role default 'member'
) returns setof uuid
language plpgsql
security definer
set search_path to ''
as $$
declare
  req_level int;
  api_key text;
  org_id uuid;
begin
  req_level := case role::text
    when 'owner' then 3
    when 'admin' then 2
    else 1 -- 'member'
  end;

  -- First, try JWT authentication via auth.uid()
  if auth.uid() is not null then
    return query select organization_id from public.agents
    where user_id = auth.uid()
    and (
      case (extra->>'role')
        when 'owner' then 3
        when 'admin' then 2
        else 1
      end
    ) >= req_level;

    if found then return; end if;
  end if;

  -- Fallback to API key authentication
  api_key := current_setting(
    'request.headers', true
  )::json->>'api-key';

  if api_key is not null then
    select a.organization_id into org_id
    from public.api_keys a
    where a.key = api_key
    and (
      case (a.role::text)
        when 'owner' then 3
        when 'admin' then 2
        else 1
      end
    ) >= req_level;

    if org_id is not null then
      return next org_id;
      return;
    end if;
  end if;

  raise exception using
    errcode = '42501',
    message = 'authentication required';
end;
$$;
```

The trick is mapping roles to numbers (`owner=3, admin=2, member=1`) so a single `>=` comparison handles the entire hierarchy. If you ask for `'admin'` access, owners get in too.

Then every RLS policy becomes a one-liner:

```sql
create policy "members can read their orgs messages"
on public.messages
for select
to authenticated, anon
using (
  organization_id in (
    select public.get_authorized_orgs('member')
  )
);
```

Notice `to authenticated, anon` — this is what makes API key auth work. The `anon` role allows unauthenticated requests (no JWT), and the function falls back to checking the `api-key` header. The function is `SECURITY DEFINER` so it can access `auth.uid()` and request headers regardless of who calls it.

One function, two auth methods, three role levels, and every table uses the same pattern.

## 2. Event-Driven Architecture: Triggers That Call Edge Functions

This is the core of OpenBSP's architecture. When a message is inserted into the database, Postgres triggers decide what happens next — depending on the message direction, status, and type.

Here's a simplified view of the trigger chain on the `messages` table:

```sql
-- Incoming message → call the AI agent
create trigger handle_incoming_message_to_agent
after insert on public.messages
for each row
when (
  new.direction = 'incoming'
  and (new.status ->> 'pending') is not null
)
execute function public.edge_function('/agent-client', 'post');

-- Outgoing message → send via WhatsApp
create trigger handle_outgoing_message_to_dispatcher
after insert on public.messages
for each row
when (
  new.direction = 'outgoing'
  and new.timestamp <= now()
  and (new.status ->> 'pending') is not null
)
execute function public.edge_function('/whatsapp-dispatcher', 'post');

-- File attachment → extract content with AI
create trigger handle_message_to_media_preprocessor
after insert on public.messages
for each row
when (
  (new.direction = 'outgoing' or new.direction = 'incoming')
  and (new.status ->> 'pending') is not null
  and (new.content ->> 'type') = 'file'
)
execute function public.edge_function('/media-preprocessor', 'post');
```

The `WHEN` clause is doing the heavy lifting here. It acts as a router — each trigger only fires for its specific combination of direction, status, and content type. No if/else logic in the Edge Functions, no message bus, no queue. Just Postgres doing what it does best.

The `edge_function()` is a reusable trigger function that calls any Edge Function:

```sql
create function public.edge_function() returns trigger
language plpgsql
security definer
as $$
declare
  request_id bigint;
  payload jsonb;
  base_url text;
  auth_token text;
  path text := tg_argv[0]::text;   -- e.g., '/agent-client'
  method text := tg_argv[1]::text;  -- 'post' or 'get'
begin
  -- Credentials stored in Supabase Vault
  select decrypted_secret into base_url
  from vault.decrypted_secrets
  where name = 'edge_functions_url';

  select decrypted_secret into auth_token
  from vault.decrypted_secrets
  where name = 'edge_functions_token';

  payload = jsonb_build_object(
    'record', new,
    'type', tg_op
  );

  select http_post into request_id from net.http_post(
    base_url || path,
    payload,
    '{}'::jsonb,
    jsonb_build_object(
      'content-type', 'application/json',
      'authorization', 'Bearer ' || auth_token
    ),
    10000  -- timeout_ms
  );

  return new;
end;
$$;
```

We use `tg_argv` (trigger arguments) to pass the Edge Function path and HTTP method. One function handles all invocations — you just configure different triggers with different arguments.

The `net.http_post` call is asynchronous (`pg_net` extension), so the INSERT returns immediately. The Edge Function runs in the background — you don't want a slow AI response blocking your database transaction.

The credentials are stored in [Supabase Vault](https://supabase.com/docs/guides/database/vault), not as environment variables. Triggers don't have access to Edge Function env vars, so Vault is the way to go.

## 3. NoSQL-Like JSON Merging in Postgres

Here's a problem we hit early on. A message has a `status` column that looks like this:

```json
{"pending": "2024-01-01T10:00:00Z"}
```

When the message is sent, we want to add a key:

```json
{"pending": "2024-01-01T10:00:00Z", "sent": "2024-01-01T10:00:05Z"}
```

But Supabase's default behavior on UPDATE replaces the entire column. If we do:

```sql
update messages
set status = '{"sent": "2024-01-01T10:00:05Z"}'
where id = '...';
```

We lose the `pending` timestamp. Gone. In MongoDB or Firestore, you'd use something like `$set` or `updateDoc` to merge fields. Postgres doesn't have that out of the box.

We built a recursive merge function:

```sql
create function public.merge_update_jsonb(
  target jsonb, path text[], object jsonb
) returns jsonb
language plpgsql
immutable
set search_path to ''
as $$
declare
  key text;
  value jsonb;
begin
  if target is null then
    target := '{}'::jsonb;
  end if;

  case jsonb_typeof(object)
    when null then
      target := null;
    when 'object' then
      -- Recurse into each key
      for key, value in select * from jsonb_each(object) loop
        target := public.merge_update_jsonb(
          target, array_append(path, key), value
        );
      end loop;
    else
      -- Scalars replace at path
      target := jsonb_set(target, path, object, true);
  end case;

  return target;
end;
$$;
```

The behavior is:

| Value type | What happens |
|------------|-------------|
| `null` | Replaces entire target with null |
| `{}` (empty object) | Recursively merges (no-op if empty) |
| Non-empty object | Recursively merges nested keys |
| String/Number/Boolean | Replaces value at that path |

Then we wrap it in a generic `BEFORE UPDATE` trigger. It takes the column name as argument, merges `OLD.column` with `NEW.column`, and writes the result back into `NEW`:

```sql
create function public.merge_update() returns trigger ...
-- Takes tg_argv[0] as column name
-- Calls merge_update_jsonb(OLD.column, NEW.column)
-- Sets NEW.column = merged result
```

Attach it to any column that needs merge semantics:

```sql
create trigger set_status
before update on public.messages
for each row
when (new.status is not null)
execute function public.merge_update('status');

create trigger set_message
before update on public.messages
for each row
when (new.content is not null)
execute function public.merge_update('content');
```

Now when we update status, new keys are added and existing keys are preserved. Document-store semantics in a relational database, with no application-level code.

## 4. CI/CD Without Supabase Branching

Supabase has a [branching feature](https://supabase.com/docs/guides/deployment/branching) for preview environments. It's great, but it's a paid feature. We needed separate staging and production environments with automated deployments, and we needed it for free.

The solution: two Supabase projects + GitHub Environments + one workflow.

```yaml
name: Release

on:
  push:
    branches:
      - main
      - develop
  workflow_dispatch:

jobs:
  db-release:
    runs-on: ubuntu-latest

    # This is the key line — same workflow, different target
    environment: ${{ github.ref == 'refs/heads/main'
      && 'Production' || 'Staging' }}

    env:
      SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
      SUPABASE_DB_PASSWORD: ${{ secrets.SUPABASE_DB_PASSWORD }}
      SUPABASE_PROJECT_ID: ${{ vars.SUPABASE_PROJECT_ID }}

    steps:
      - uses: actions/checkout@v4
      - uses: supabase/setup-cli@v1

      - run: supabase link --project-ref $SUPABASE_PROJECT_ID
      - run: supabase db push
      - run: supabase functions deploy
```

The `environment` line does all the magic. Push to `develop` → deploys to Staging (Supabase project A). Push to `main` → deploys to Production (Supabase project B). Each GitHub Environment has its own set of secrets and variables pointing to a different Supabase project.

`supabase db push` applies pending migrations. `supabase functions deploy` deploys all Edge Functions. One workflow, two environments, zero branching fees.

## 5. Declarative Schemas: The Source of Truth

Traditional Supabase projects use migrations as the source of truth. Over time, you end up with dozens of migration files, and understanding the current state of the database requires reading them all in order.

We took a different approach. Instead of writing migrations by hand, we maintain **schema files** that represent the desired state of the database:

```
supabase/schemas/
  00_extensions.sql
  01_types.sql
  02_functions/
    02-01_json_utils.sql
    02-02_edge_functions.sql
    02-03_trigger_functions.sql
    02-04_rcp_functions.sql
  03_models/
    03-00_organizations.sql
    03-01_organizations_addresses.sql
    03-02_contacts.sql
    03-03_conversations.sql
    03-04_agents.sql
    03-05_messages.sql
    ...
  04_functions_post_tables/
    04-01_auth_helpers.sql
  05_rls/
    05-00_organizations_rls.sql
    05-05_messages_rls.sql
    ...
  06_billing/
    ...
```

The numbered prefixes enforce execution order: extensions first, then types, then functions (that don't reference tables), then tables, then functions that DO reference tables, then RLS policies. This ordering matters because Postgres needs to see types before functions that use them, and tables before RLS policies.

The workflow is:

1. Edit a schema file (e.g., add a column to `03-05_messages.sql`)
2. Run `supabase db diff -f my_change` — Supabase compares the schema files against the running database and generates a migration
3. Review and commit the migration
4. `supabase db push` applies it

You never write migration SQL by hand — you describe what you want, and the diff tool figures out how to get there. When you review a PR, you see the final state of the table, not a chain of ALTER statements.

---

These patterns emerged from building a real product on Supabase over 3 years. Some I planned, some I stumbled into. All of them are running in production at [Mirlo](https://mirlo.com).

If you want to see them in context:

- **API repo**: [github.com/matiasbattocchia/open-bsp-api](https://github.com/matiasbattocchia/open-bsp-api)
- **UI repo**: [github.com/matiasbattocchia/open-bsp-ui](https://github.com/matiasbattocchia/open-bsp-ui)
- **Live demo**: [web.openbsp.dev](https://web.openbsp.dev)

Happy to discuss any of these — some things worked great, some I had to work around. What patterns have you found useful in your Supabase projects?
