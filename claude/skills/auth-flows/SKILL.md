---
name: auth-flows
description: >
  Authentication and authorization patterns for JobBridge. Two user roles: client (US job seeker)
  and worker (Indian student). Always read this before writing signup, login, onboarding,
  role checks, profile setup, or any protected route logic. Trigger on any mention of:
  auth, login, signup, session, user, role, client vs worker, onboarding, protected routes,
  or permissions.
---

# Auth Flows — JobBridge

## Two User Types

| Role | Description | Key Permissions |
|---|---|---|
| `client` | US job seeker | Post contracts, hire workers, approve submissions, release payments |
| `worker` | Indian/international student | Browse contracts, apply, submit daily work, receive payouts |

---

## Signup Flow

### Step 1: Signup page (`/signup`)
User enters: name, email, password, and **selects their role** (Client or Worker).

```tsx
// app/(auth)/signup/page.tsx
'use client'
export default function SignupPage() {
  const [role, setRole] = useState<'client' | 'worker'>('client')
  // ...
  async function handleSignup(e) {
    e.preventDefault()
    const supabase = createBrowserClient()
    await supabase.auth.signUp({
      email,
      password,
      options: {
        data: { full_name: name, role }
      }
    })
    router.push('/onboarding')
  }
}
```

### Step 2: Onboarding (`/onboarding`)
After email confirmation, user completes their profile based on their role.

**Client onboarding collects:**
- Country (pre-filled: United States)
- LinkedIn URL
- Resume upload (PDF)
- Target job roles (tags input)
- Target locations
- Preferred job boards (checkboxes: LinkedIn, Indeed, Glassdoor, ZipRecruiter, etc.)

**Worker onboarding collects:**
- Country
- University name
- Graduation year
- Skills (tags input)
- Portfolio/GitHub URL
- Available hours per day (slider: 1–8)
- Bank account details for Razorpay payouts

### After onboarding:
- Insert into `client_profiles` or `worker_profiles`
- Redirect to role-specific dashboard

---

## Login Flow

Standard email/password with Supabase:

```tsx
async function handleLogin(email: string, password: string) {
  const supabase = createBrowserClient()
  const { error } = await supabase.auth.signInWithPassword({ email, password })
  if (error) throw new Error(error.message)

  // Redirect based on role
  const { data: { user } } = await supabase.auth.getUser()
  const { data: profile } = await supabase
    .from('profiles')
    .select('role')
    .eq('id', user!.id)
    .single()

  router.push(profile?.role === 'client' ? '/client/dashboard' : '/worker/dashboard')
}
```

---

## Route Protection

### Middleware protects all routes

```ts
// middleware.ts — see nextjs-conventions skill
// Unauthenticated users → /login
// Authenticated users on wrong role routes → redirect to their dashboard
```

### Role guard in layouts

```tsx
// app/(client)/layout.tsx
export default async function ClientLayout({ children }) {
  const supabase = createServerClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) redirect('/login')

  const { data: profile } = await supabase
    .from('profiles').select('role').eq('id', user.id).single()

  if (profile?.role !== 'client') redirect('/worker/dashboard')

  return <>{children}</>
}
```

```tsx
// app/(worker)/layout.tsx — same pattern but checks role === 'worker'
```

---

## Checking Auth in Server Actions

Always use this pattern at the top of every Server Action:

```ts
'use server'
export async function someAction(data: SomeType) {
  const supabase = createServerClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) throw new Error('Unauthorized')

  // optionally check role
  const { data: profile } = await supabase
    .from('profiles').select('role').eq('id', user.id).single()
  if (profile?.role !== 'client') throw new Error('Forbidden')

  // proceed with action
}
```

---

## Sign Out

```tsx
'use client'
async function handleSignOut() {
  const supabase = createBrowserClient()
  await supabase.auth.signOut()
  router.push('/login')
}
```

---

## Profile Completeness Check

After login, check if the user has completed onboarding:

```ts
export async function getOnboardingStatus(userId: string, role: string) {
  const supabase = createServerClient()

  if (role === 'client') {
    const { data } = await supabase
      .from('client_profiles')
      .select('id, resume_url')
      .eq('id', userId)
      .single()
    return { complete: !!data?.resume_url }
  }

  if (role === 'worker') {
    const { data } = await supabase
      .from('worker_profiles')
      .select('id, razorpay_fund_account_id')
      .eq('id', userId)
      .single()
    return { complete: !!data?.razorpay_fund_account_id }
  }

  return { complete: false }
}
```

If not complete, redirect to `/onboarding` from the dashboard layout.

---

## Session in Client Components

```tsx
'use client'
import { createBrowserClient } from '@/lib/supabase/client'
import { useEffect, useState } from 'react'
import type { User } from '@supabase/supabase-js'

export function useUser() {
  const supabase = createBrowserClient()
  const [user, setUser] = useState<User | null>(null)

  useEffect(() => {
    supabase.auth.getUser().then(({ data }) => setUser(data.user))

    const { data: { subscription } } = supabase.auth.onAuthStateChange(
      (_, session) => setUser(session?.user ?? null)
    )
    return () => subscription.unsubscribe()
  }, [])

  return user
}
```

---

## Google OAuth (optional, add later)

```ts
await supabase.auth.signInWithOAuth({
  provider: 'google',
  options: {
    redirectTo: `${process.env.NEXT_PUBLIC_APP_URL}/auth/callback`,
    queryParams: { access_type: 'offline', prompt: 'consent' }
  }
})
```

Requires callback route at `/app/auth/callback/route.ts` using `exchangeCodeForSession`.
