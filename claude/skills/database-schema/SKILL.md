---
name: database-schema
description: >
  Complete PostgreSQL/Supabase schema for the JobBridge app. Always read this skill before
  writing any database query, creating a migration, modifying a table, adding a column,
  writing RLS policies, or designing any API that touches the database. Trigger on any mention
  of: tables, columns, schema, migration, SQL, database design, relations, foreign keys, or
  any entity like contracts, submissions, payments, messages, users, profiles, escrow.
---

# Database Schema — JobBridge

> Read this before ANY database operation. All table definitions, relationships, and RLS rules are here.

---

## Supabase Setup Notes

- Auth is handled by `supabase.auth.users` (built-in). We extend it with a `profiles` table.
- Always use `supabase.auth.getUser()` server-side for secure auth checks.
- Enable Row Level Security (RLS) on every table.
- Use `supabase/migrations/` folder for all schema changes.

---

## Tables

### `profiles`
Extends Supabase auth users. Created automatically on signup via trigger.

```sql
create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  role text not null check (role in ('client', 'worker')),
  full_name text not null,
  avatar_url text,
  country text,
  bio text,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);
```

### `client_profiles`
Extra info for clients.

```sql
create table client_profiles (
  id uuid primary key references profiles(id) on delete cascade,
  resume_url text,
  linkedin_url text,
  target_roles text[],          -- e.g. ['Software Engineer', 'Data Analyst']
  target_locations text[],      -- e.g. ['Remote', 'New York']
  preferred_job_boards text[]   -- e.g. ['LinkedIn', 'Indeed', 'Glassdoor']
);
```

### `worker_profiles`
Extra info for workers.

```sql
create table worker_profiles (
  id uuid primary key references profiles(id) on delete cascade,
  university text,
  graduation_year int,
  skills text[],
  portfolio_url text,
  razorpay_contact_id text,     -- for payouts
  razorpay_fund_account_id text,
  available_hours_per_day int default 3
);
```

### `contracts`
Job-application contracts posted by clients.

```sql
create table contracts (
  id uuid primary key default gen_random_uuid(),
  client_id uuid not null references profiles(id),
  worker_id uuid references profiles(id),  -- null until hired
  title text not null,                     -- e.g. "Apply to SWE jobs daily"
  description text not null,
  applications_per_day int not null,
  duration_days int not null,
  start_date date,
  end_date date,
  daily_rate_usd numeric(10,2) not null,
  total_amount_usd numeric(10,2) generated always as (daily_rate_usd * duration_days) stored,
  status text not null default 'open'
    check (status in ('open','in_review','active','completed','cancelled','disputed')),
  special_instructions text,
  job_boards text[],
  target_roles text[],
  target_locations text[],
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);
```

**Status lifecycle:** `open` → `in_review` (worker applied) → `active` (worker hired + funded) → `completed` / `cancelled` / `disputed`

### `contract_applications`
Worker bids on a contract.

```sql
create table contract_applications (
  id uuid primary key default gen_random_uuid(),
  contract_id uuid not null references contracts(id) on delete cascade,
  worker_id uuid not null references profiles(id),
  pitch text not null,
  status text not null default 'pending'
    check (status in ('pending','accepted','rejected')),
  created_at timestamptz default now(),
  unique(contract_id, worker_id)
);
```

### `escrow_accounts`
Holds the full contract payment. One per contract.

```sql
create table escrow_accounts (
  id uuid primary key default gen_random_uuid(),
  contract_id uuid not null unique references contracts(id),
  total_amount_usd numeric(10,2) not null,
  released_amount_usd numeric(10,2) default 0,
  remaining_amount_usd numeric(10,2) generated always as (total_amount_usd - released_amount_usd) stored,
  stripe_payment_intent_id text,
  status text not null default 'pending'
    check (status in ('pending','funded','partially_released','fully_released','refunded')),
  funded_at timestamptz,
  created_at timestamptz default now()
);
```

### `daily_submissions`
Worker's daily work report.

```sql
create table daily_submissions (
  id uuid primary key default gen_random_uuid(),
  contract_id uuid not null references contracts(id),
  worker_id uuid not null references profiles(id),
  submission_date date not null,
  jobs_applied jsonb not null default '[]',
  -- shape: [{ company, role, job_url, board, applied_at, status }]
  screenshot_urls text[],
  notes text,
  status text not null default 'pending'
    check (status in ('pending','approved','revision_requested','disputed')),
  reviewed_at timestamptz,
  auto_approved boolean default false,
  created_at timestamptz default now(),
  unique(contract_id, submission_date)
);
```

### `payments`
Individual payment releases from escrow to worker.

```sql
create table payments (
  id uuid primary key default gen_random_uuid(),
  contract_id uuid not null references contracts(id),
  submission_id uuid references daily_submissions(id),
  worker_id uuid not null references profiles(id),
  client_id uuid not null references profiles(id),
  gross_amount_usd numeric(10,2) not null,   -- daily_rate_usd
  platform_fee_usd numeric(10,2) not null,   -- 10%
  net_amount_usd numeric(10,2) not null,     -- gross - fee
  inr_equivalent numeric(10,2),              -- calculated at payout time
  stripe_transfer_id text,
  razorpay_payout_id text,
  status text not null default 'pending'
    check (status in ('pending','processing','completed','failed')),
  created_at timestamptz default now()
);
```

### `messages`
In-app chat between client and worker on a contract.

```sql
create table messages (
  id uuid primary key default gen_random_uuid(),
  contract_id uuid not null references contracts(id),
  sender_id uuid not null references profiles(id),
  content text not null,
  is_read boolean default false,
  created_at timestamptz default now()
);
```

### `reviews`
End-of-contract rating (both directions).

```sql
create table reviews (
  id uuid primary key default gen_random_uuid(),
  contract_id uuid not null references contracts(id),
  reviewer_id uuid not null references profiles(id),
  reviewee_id uuid not null references profiles(id),
  rating int not null check (rating between 1 and 5),
  comment text,
  created_at timestamptz default now(),
  unique(contract_id, reviewer_id)
);
```

---

## Key Indexes

```sql
create index on contracts(client_id);
create index on contracts(worker_id);
create index on contracts(status);
create index on daily_submissions(contract_id);
create index on daily_submissions(submission_date);
create index on messages(contract_id);
create index on messages(created_at);
create index on payments(worker_id);
create index on payments(contract_id);
```

---

## Auto-profile Creation Trigger

```sql
create or replace function handle_new_user()
returns trigger as $$
begin
  insert into profiles (id, full_name, role)
  values (
    new.id,
    coalesce(new.raw_user_meta_data->>'full_name', 'User'),
    coalesce(new.raw_user_meta_data->>'role', 'client')
  );
  return new;
end;
$$ language plpgsql security definer;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure handle_new_user();
```

---

## Auto-approve Submissions (24h rule)

Run this as a Supabase Edge Function on a cron schedule (every hour):

```sql
update daily_submissions
set status = 'approved', auto_approved = true, reviewed_at = now()
where status = 'pending'
  and created_at < now() - interval '24 hours';
```

---

## RLS Policies

```sql
-- profiles: users can read all, update only own
alter table profiles enable row level security;
create policy "Public profiles readable" on profiles for select using (true);
create policy "Own profile updatable" on profiles for update using (auth.uid() = id);

-- contracts: open ones readable by all; client owns their own
alter table contracts enable row level security;
create policy "Open contracts readable" on contracts for select using (status = 'open' or client_id = auth.uid() or worker_id = auth.uid());
create policy "Client creates contract" on contracts for insert with check (auth.uid() = client_id);
create policy "Client updates own contract" on contracts for update using (auth.uid() = client_id);

-- messages: only participants of a contract can see messages
alter table messages enable row level security;
create policy "Contract participants see messages" on messages for select
  using (
    auth.uid() in (
      select client_id from contracts where id = contract_id
      union
      select worker_id from contracts where id = contract_id
    )
  );
create policy "Participants send messages" on messages for insert
  with check (
    auth.uid() = sender_id and
    auth.uid() in (
      select client_id from contracts where id = contract_id
      union
      select worker_id from contracts where id = contract_id
    )
  );

-- daily_submissions: client and worker of contract can see; worker inserts
alter table daily_submissions enable row level security;
create policy "Participants view submissions" on daily_submissions for select
  using (
    auth.uid() in (
      select client_id from contracts where id = contract_id
      union
      select worker_id from contracts where id = contract_id
    )
  );
create policy "Worker submits daily report" on daily_submissions for insert
  with check (auth.uid() = worker_id);
create policy "Client approves submission" on daily_submissions for update
  using (auth.uid() in (select client_id from contracts where id = contract_id));

-- payments: parties of the contract can view
alter table payments enable row level security;
create policy "Parties view payments" on payments for select
  using (auth.uid() = client_id or auth.uid() = worker_id);
```
