---
name: nextjs-conventions
description: >
  Coding conventions, patterns, and architecture rules for the JobBridge Next.js 14 App Router
  codebase. Always read this before writing any Next.js page, component, API route, server action,
  or layout. Trigger on any mention of: pages, components, routes, API, server actions, layouts,
  loading states, error handling, TypeScript types, hooks, middleware, or any frontend/backend code.
---

# Next.js 14 Conventions — JobBridge

> Always follow these patterns. Do not deviate. Consistency is critical for Claude Code to work reliably across the codebase.

---

## App Router Structure

```
/app
  layout.tsx              ← root layout (fonts, providers)
  page.tsx                ← landing page (public)
  /(auth)/
    layout.tsx            ← auth layout (no nav)
    login/page.tsx
    signup/page.tsx
    onboarding/page.tsx   ← role + profile setup after signup
  /(client)/
    layout.tsx            ← client layout (client nav)
    dashboard/page.tsx
    post-contract/page.tsx
    contracts/
      page.tsx            ← list of client's contracts
      [id]/page.tsx       ← contract detail
    submissions/[id]/page.tsx  ← review daily submission
  /(worker)/
    layout.tsx            ← worker layout (worker nav)
    dashboard/page.tsx
    browse/page.tsx       ← open contracts
    my-contracts/page.tsx
    submit/[contractId]/page.tsx
  /(shared)/
    messages/[contractId]/page.tsx
    profile/[userId]/page.tsx
  /api/
    webhooks/
      stripe/route.ts
      razorpay/route.ts
```

---

## Component Rules

### Server vs Client Components

**Default to Server Components.** Only add `'use client'` when you need:
- `useState`, `useEffect`, `useReducer`
- Event listeners (`onClick`, `onChange`)
- Browser APIs
- Realtime subscriptions

```tsx
// ✅ Server Component (default)
export default async function ContractList() {
  const contracts = await getContracts()
  return <div>{contracts.map(c => <ContractCard key={c.id} contract={c} />)}</div>
}

// ✅ Client Component (only when needed)
'use client'
export function SubmitButton({ onClick }: { onClick: () => void }) {
  return <button onClick={onClick}>Submit</button>
}
```

### Component File Naming

- Pages: `page.tsx` (Next.js convention)
- Layouts: `layout.tsx`
- Components: `PascalCase.tsx` (e.g., `ContractCard.tsx`)
- Hooks: `camelCase.ts` prefixed with `use` (e.g., `useContracts.ts`)
- Utils: `camelCase.ts` (e.g., `formatCurrency.ts`)

### Component Location

| Type | Location |
|---|---|
| shadcn/ui components | `/components/ui/` |
| Client-only components | `/components/client/` |
| Worker-only components | `/components/worker/` |
| Shared components | `/components/shared/` |
| Page-specific | Colocate in page folder if not reused |

---

## Data Fetching Patterns

### Server-side data fetching (preferred)

```tsx
// lib/supabase/server.ts handles the client
import { createServerClient } from '@/lib/supabase/server'

async function getContract(id: string) {
  const supabase = createServerClient()
  const { data, error } = await supabase
    .from('contracts')
    .select('*, profiles(*), daily_submissions(*)')
    .eq('id', id)
    .single()

  if (error) throw new Error(error.message)
  return data
}
```

### Server Actions (for mutations)

Always use server actions for create/update/delete. Never call Supabase from client components directly.

```tsx
// app/(client)/post-contract/actions.ts
'use server'
import { createServerClient } from '@/lib/supabase/server'
import { revalidatePath } from 'next/cache'

export async function createContract(formData: FormData) {
  const supabase = createServerClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) throw new Error('Unauthorized')

  const { error } = await supabase.from('contracts').insert({
    client_id: user.id,
    title: formData.get('title') as string,
    // ...
  })

  if (error) throw new Error(error.message)
  revalidatePath('/client/contracts')
}
```

### API Routes (for webhooks only)

Use `/app/api/` routes ONLY for:
- Stripe webhooks
- Razorpay webhooks
- Any external service callback

Everything else uses Server Actions.

---

## TypeScript Types

Define all types in `/types/index.ts`. Always use types, never `any`.

```ts
// types/index.ts

export type UserRole = 'client' | 'worker'

export type ContractStatus =
  | 'open' | 'in_review' | 'active'
  | 'completed' | 'cancelled' | 'disputed'

export type SubmissionStatus =
  | 'pending' | 'approved' | 'revision_requested' | 'disputed'

export interface Profile {
  id: string
  role: UserRole
  full_name: string
  avatar_url: string | null
  country: string | null
  bio: string | null
  created_at: string
}

export interface Contract {
  id: string
  client_id: string
  worker_id: string | null
  title: string
  description: string
  applications_per_day: number
  duration_days: number
  daily_rate_usd: number
  total_amount_usd: number
  status: ContractStatus
  special_instructions: string | null
  job_boards: string[]
  target_roles: string[]
  created_at: string
}

export interface DailySubmission {
  id: string
  contract_id: string
  worker_id: string
  submission_date: string
  jobs_applied: JobEntry[]
  screenshot_urls: string[]
  notes: string | null
  status: SubmissionStatus
  auto_approved: boolean
  created_at: string
}

export interface JobEntry {
  company: string
  role: string
  job_url: string
  board: string
  applied_at: string
  status: 'applied' | 'responded' | 'rejected'
}

export interface Message {
  id: string
  contract_id: string
  sender_id: string
  content: string
  is_read: boolean
  created_at: string
}
```

---

## Loading & Error States

Every page that fetches data must have:

```
/app/(client)/contracts/
  page.tsx        ← the page
  loading.tsx     ← skeleton UI shown during fetch
  error.tsx       ← error boundary
```

```tsx
// loading.tsx
export default function Loading() {
  return <div className="space-y-4">
    {[...Array(3)].map((_, i) => (
      <div key={i} className="h-24 rounded-lg bg-muted animate-pulse" />
    ))}
  </div>
}

// error.tsx
'use client'
export default function Error({ error, reset }: { error: Error, reset: () => void }) {
  return (
    <div className="text-center py-12">
      <p className="text-destructive">{error.message}</p>
      <button onClick={reset} className="mt-4 underline">Try again</button>
    </div>
  )
}
```

---

## Middleware (Auth Protection)

```ts
// middleware.ts
import { createMiddlewareClient } from '@supabase/auth-helpers-nextjs'
import { NextResponse } from 'next/server'

export async function middleware(req) {
  const res = NextResponse.next()
  const supabase = createMiddlewareClient({ req, res })
  const { data: { session } } = await supabase.auth.getSession()

  const isAuthRoute = req.nextUrl.pathname.startsWith('/login') ||
                      req.nextUrl.pathname.startsWith('/signup')

  if (!session && !isAuthRoute && req.nextUrl.pathname !== '/') {
    return NextResponse.redirect(new URL('/login', req.url))
  }

  return res
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|api/webhooks).*)']
}
```

---

## Environment Variables

Access via `process.env`. Never expose server-side keys to client.

- `NEXT_PUBLIC_*` → safe for client
- All others → server-only

Always validate env vars at startup in `/lib/env.ts`:

```ts
export const env = {
  supabaseUrl: process.env.NEXT_PUBLIC_SUPABASE_URL!,
  supabaseAnonKey: process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  stripeSecretKey: process.env.STRIPE_SECRET_KEY!,
  razorpayKeyId: process.env.RAZORPAY_KEY_ID!,
  razorpayKeySecret: process.env.RAZORPAY_KEY_SECRET!,
  platformFeePercent: Number(process.env.PLATFORM_FEE_PERCENT ?? 10),
}
```

---

## Utility Functions

```ts
// lib/utils/currency.ts
export function formatUSD(amount: number) {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(amount)
}

export function formatINR(amount: number) {
  return new Intl.NumberFormat('en-IN', { style: 'currency', currency: 'INR' }).format(amount)
}

export function calculatePlatformFee(amount: number, feePercent = 10) {
  return parseFloat((amount * feePercent / 100).toFixed(2))
}

export function calculateNetPayout(grossAmount: number, feePercent = 10) {
  const fee = calculatePlatformFee(grossAmount, feePercent)
  return parseFloat((grossAmount - fee).toFixed(2))
}
```
