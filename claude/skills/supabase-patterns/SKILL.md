---
name: supabase-patterns
description: >
  All Supabase usage patterns for the JobBridge app: client setup, auth, database queries,
  realtime subscriptions, file storage, and edge functions. Always read this before writing
  any Supabase code, auth logic, database call, file upload, or realtime feature.
  Trigger on any mention of: Supabase, database query, auth, session, RLS, storage,
  realtime, subscription, or any CRUD operation on app data.
---

# Supabase Patterns — JobBridge

---

## Client Setup

### Server-side client (for Server Components, Server Actions, API routes)

```ts
// lib/supabase/server.ts
import { createServerComponentClient } from '@supabase/auth-helpers-nextjs'
import { cookies } from 'next/headers'
import type { Database } from '@/types/supabase'

export function createServerClient() {
  return createServerComponentClient<Database>({ cookies })
}
```

### Client-side client (for Client Components only)

```ts
// lib/supabase/client.ts
import { createClientComponentClient } from '@supabase/auth-helpers-nextjs'
import type { Database } from '@/types/supabase'

export function createBrowserClient() {
  return createClientComponentClient<Database>()
}
```

### Generate TypeScript types from Supabase

Run this whenever the schema changes:
```bash
npx supabase gen types typescript --project-id YOUR_PROJECT_ID > types/supabase.ts
```

---

## Auth Patterns

### Sign up with role

```ts
// Always pass role in metadata so the trigger creates the profile correctly
const { error } = await supabase.auth.signUp({
  email,
  password,
  options: {
    data: {
      full_name: name,
      role: 'client' // or 'worker'
    }
  }
})
```

### Get current user (server-side — ALWAYS use this, not getSession)

```ts
// In Server Components and Server Actions
const supabase = createServerClient()
const { data: { user }, error } = await supabase.auth.getUser()
if (!user) redirect('/login')
```

### Get user's role

```ts
async function getUserRole(userId: string) {
  const supabase = createServerClient()
  const { data } = await supabase
    .from('profiles')
    .select('role')
    .eq('id', userId)
    .single()
  return data?.role
}
```

### Protect a Server Action by role

```ts
'use server'
async function clientOnlyAction() {
  const supabase = createServerClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) throw new Error('Unauthorized')

  const role = await getUserRole(user.id)
  if (role !== 'client') throw new Error('Forbidden: clients only')

  // proceed...
}
```

---

## Common Query Patterns

### Fetch contract with all relations

```ts
const { data: contract } = await supabase
  .from('contracts')
  .select(`
    *,
    client:profiles!client_id(id, full_name, avatar_url),
    worker:profiles!worker_id(id, full_name, avatar_url),
    daily_submissions(*)
  `)
  .eq('id', contractId)
  .single()
```

### Fetch open contracts (for worker browse page)

```ts
const { data: contracts } = await supabase
  .from('contracts')
  .select(`
    *,
    client:profiles!client_id(full_name, avatar_url, country)
  `)
  .eq('status', 'open')
  .order('created_at', { ascending: false })
```

### Fetch worker's active contracts

```ts
const { data: contracts } = await supabase
  .from('contracts')
  .select('*, client:profiles!client_id(*)')
  .eq('worker_id', user.id)
  .eq('status', 'active')
```

### Check if worker already submitted today

```ts
const today = new Date().toISOString().split('T')[0]
const { data: existing } = await supabase
  .from('daily_submissions')
  .select('id')
  .eq('contract_id', contractId)
  .eq('submission_date', today)
  .single()

if (existing) throw new Error('Already submitted today')
```

### Get payment history for a worker

```ts
const { data: payments } = await supabase
  .from('payments')
  .select('*, contracts(title)')
  .eq('worker_id', workerId)
  .order('created_at', { ascending: false })
```

---

## Realtime (In-App Chat)

Use Supabase Realtime to stream new messages in the chat view.

```tsx
// components/shared/ChatWindow.tsx
'use client'
import { useEffect, useState } from 'react'
import { createBrowserClient } from '@/lib/supabase/client'
import type { Message } from '@/types'

export function ChatWindow({ contractId, currentUserId }: {
  contractId: string
  currentUserId: string
}) {
  const supabase = createBrowserClient()
  const [messages, setMessages] = useState<Message[]>([])

  useEffect(() => {
    // Initial load
    supabase
      .from('messages')
      .select('*')
      .eq('contract_id', contractId)
      .order('created_at')
      .then(({ data }) => setMessages(data ?? []))

    // Subscribe to new messages
    const channel = supabase
      .channel(`messages:${contractId}`)
      .on('postgres_changes', {
        event: 'INSERT',
        schema: 'public',
        table: 'messages',
        filter: `contract_id=eq.${contractId}`
      }, (payload) => {
        setMessages(prev => [...prev, payload.new as Message])
      })
      .subscribe()

    return () => { supabase.removeChannel(channel) }
  }, [contractId])

  return (
    <div>
      {messages.map(msg => (
        <div key={msg.id} className={msg.sender_id === currentUserId ? 'text-right' : 'text-left'}>
          {msg.content}
        </div>
      ))}
    </div>
  )
}
```

### Send a message (Server Action)

```ts
'use server'
export async function sendMessage(contractId: string, content: string) {
  const supabase = createServerClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) throw new Error('Unauthorized')

  const { error } = await supabase.from('messages').insert({
    contract_id: contractId,
    sender_id: user.id,
    content
  })
  if (error) throw new Error(error.message)
}
```

---

## File Storage (Resumes & Screenshots)

### Upload resume (client profile)

```ts
export async function uploadResume(file: File, userId: string) {
  const supabase = createBrowserClient()
  const path = `resumes/${userId}/${file.name}`

  const { error } = await supabase.storage
    .from('user-files')
    .upload(path, file, { upsert: true })

  if (error) throw new Error(error.message)

  const { data } = supabase.storage.from('user-files').getPublicUrl(path)
  return data.publicUrl
}
```

### Upload daily submission screenshots

```ts
export async function uploadScreenshots(files: File[], submissionId: string) {
  const supabase = createBrowserClient()
  const urls: string[] = []

  for (const file of files) {
    const path = `submissions/${submissionId}/${file.name}`
    await supabase.storage.from('user-files').upload(path, file)
    const { data } = supabase.storage.from('user-files').getPublicUrl(path)
    urls.push(data.publicUrl)
  }

  return urls
}
```

### Supabase Storage Buckets to Create

| Bucket | Public? | Contents |
|---|---|---|
| `user-files` | No (use signed URLs) | Resumes, screenshots |
| `avatars` | Yes | Profile photos |

---

## Edge Functions (Cron Jobs)

### Auto-approve submissions after 24h
Deploy as a Supabase Edge Function with a daily cron schedule.

```ts
// supabase/functions/auto-approve-submissions/index.ts
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

Deno.serve(async () => {
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  )

  const { error } = await supabase
    .from('daily_submissions')
    .update({ status: 'approved', auto_approved: true, reviewed_at: new Date().toISOString() })
    .eq('status', 'pending')
    .lt('created_at', new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString())

  return new Response(JSON.stringify({ success: !error }), {
    headers: { 'Content-Type': 'application/json' }
  })
})
```

Schedule in `supabase/config.toml`:
```toml
[functions.auto-approve-submissions]
schedule = "0 * * * *"  # every hour
```
