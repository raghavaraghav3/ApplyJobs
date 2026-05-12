---
name: project-overview
description: >
  Master reference for the JobBridge app — a student freelance marketplace where US job seekers
  post daily job-application tasks and Indian/international students earn by completing them.
  Always read this skill first before any feature work, API design, database change, or UI task.
  Trigger on any mention of: the app, users, hirers, workers, job posts, contracts, tasks, payments,
  escrow, daily releases, or any new feature discussion.
---

# JobBridge — Project Overview

## What This App Does

JobBridge is a two-sided marketplace that connects:

- **Clients (US Job Seekers)** — people in the US looking for jobs but unable to spend hours applying daily. They post a contract specifying what they need done every day.
- **Workers (Indian/International Students)** — 3rd/4th year students with 2–3 free hours per day who apply to jobs on behalf of clients and earn part-time income.

A typical use case:
> A US student posts: "Apply to 10 software engineering jobs per day on my behalf using my resume. Duration: 1 month. Budget: $60 total ($2/day)."
> An Indian student applies for this contract, gets hired, applies to jobs daily, submits proof, and the client releases that day's payment upon approval.

---

## User Roles

### Client
- Based in the US (primarily)
- Posts job-application contracts
- Uploads their resume, LinkedIn, job preferences
- Reviews daily work submissions from workers
- Releases payment per day (escrow system)
- Can chat with their hired worker

### Worker
- Based in India or other countries
- Browses open contracts
- Applies to take on a contract
- Submits daily proof of work (screenshots, list of job links applied to)
- Receives payment after client approval
- Can chat with their client

---

## Core User Flows

### Flow 1: Client Posts a Contract
1. Client signs up → completes profile (name, location, resume upload)
2. Client creates a contract:
   - Job title/role they're targeting
   - Number of applications per day
   - Contract duration (1 week / 2 weeks / 1 month / custom)
   - Daily budget (USD)
   - Total budget = daily budget × number of days
   - Special instructions (job boards to use, keywords, locations to target)
3. Contract goes live and is visible to workers

### Flow 2: Worker Applies to a Contract
1. Worker signs up → completes profile (name, skills, education, country, portfolio)
2. Worker browses open contracts
3. Worker submits an application (short pitch + availability confirmation)
4. Client reviews applications and selects one worker
5. Contract becomes "Active" — worker is now locked in

### Flow 3: Daily Work & Payment Release
1. Each day, the worker:
   - Applies to jobs on behalf of client (using client's resume/info)
   - Submits a daily report: list of job links, company names, positions applied, any responses received
2. Client reviews the daily report
3. Client approves → that day's payment portion is released from escrow to worker
4. If client does not approve, they can request a revision or raise a dispute
5. At end of contract, any unreleased funds return to client

### Flow 4: Escrow & Payment
1. When client hires a worker, they pay the **full contract amount upfront** into escrow
2. Escrow is held by the platform
3. Each day's portion is released only after client approval
4. Worker can withdraw accumulated earnings to their bank (Razorpay for INR payouts)
5. Platform takes a % fee (e.g., 10%) from each released payment

---

## Business Rules

- A client can only have one active contract at a time per "job search campaign"
- A worker can take on up to 3 active contracts simultaneously
- Contracts must be agreed upon and funded before work begins (no work without escrow)
- Clients have 24 hours to review and approve/reject a daily submission; auto-approve after 24h
- Workers submit one report per day per contract
- Disputes go to a manual review queue (admin panel)
- Platform fee: 10% deducted from each released payment

---

## Key Entities

| Entity | Description |
|---|---|
| `User` | Base user (either client or worker) |
| `Profile` | Extended info per role |
| `Contract` | Job-application task posted by client |
| `Application` | Worker's bid to take on a contract |
| `DailySubmission` | Daily work report submitted by worker |
| `EscrowAccount` | Holds contract funds |
| `Payment` | Individual payment release record |
| `Message` | In-app chat message |
| `Review` | End-of-contract rating |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend + API | Next.js 14 (App Router) |
| Styling | Tailwind CSS + shadcn/ui |
| Database | Supabase (PostgreSQL) |
| Auth | Supabase Auth |
| File Storage | Supabase Storage |
| Realtime Chat | Supabase Realtime |
| USD Payments | Stripe |
| INR Payments/Payouts | Razorpay |
| Email | Resend |
| Deployment | Vercel |

---

## Environment Variables Needed

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
STRIPE_SECRET_KEY=
STRIPE_PUBLISHABLE_KEY=
STRIPE_WEBHOOK_SECRET=
RAZORPAY_KEY_ID=
RAZORPAY_KEY_SECRET=
RESEND_API_KEY=
NEXT_PUBLIC_APP_URL=
PLATFORM_FEE_PERCENT=10
```

---

## Folder Structure

```
/app
  /(auth)
    /login
    /signup
    /onboarding        ← role selection + profile setup
  /(client)
    /dashboard
    /post-contract
    /contracts/[id]
    /submissions/[id]  ← review daily work
  /(worker)
    /dashboard
    /browse            ← open contracts
    /my-contracts
    /submit/[contractId]
  /(shared)
    /messages/[contractId]
    /profile/[userId]
  /api
    /webhooks/stripe
    /webhooks/razorpay
/components
  /ui                  ← shadcn components
  /client              ← client-specific components
  /worker              ← worker-specific components
  /shared              ← shared components
/lib
  /supabase
  /stripe
  /razorpay
  /resend
  /utils
/hooks
/types
```
