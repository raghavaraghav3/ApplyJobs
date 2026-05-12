---
name: ui-components
description: >
  Design system, component patterns, and UI conventions for the JobBridge app.
  Always read this before building any UI, page layout, component, form, card, modal,
  or visual element. Trigger on any mention of: UI, design, components, styling, layout,
  forms, cards, buttons, colors, Tailwind, shadcn, responsive, mobile, or any visual feature.
---

# UI Components & Design System — JobBridge

## Stack

- **Tailwind CSS** — utility-first styling
- **shadcn/ui** — base components (install individually as needed)
- **lucide-react** — icons
- **Design feel:** Clean, professional, trustworthy. Think Upwork meets a modern fintech app.

---

## shadcn/ui Components to Install

Run these during project setup:

```bash
npx shadcn@latest init
npx shadcn@latest add button card badge input textarea select dialog sheet tabs avatar progress separator skeleton toast alert
```

---

## Color Palette (Tailwind config)

```ts
// tailwind.config.ts
theme: {
  extend: {
    colors: {
      brand: {
        50:  '#eff6ff',
        100: '#dbeafe',
        500: '#3b82f6',   // primary blue
        600: '#2563eb',
        700: '#1d4ed8',
        900: '#1e3a8a',
      },
      success: '#22c55e',
      warning: '#f59e0b',
      danger:  '#ef4444',
    }
  }
}
```

**Usage:**
- Primary actions: `bg-brand-600 hover:bg-brand-700 text-white`
- Destructive: `bg-danger text-white`
- Success state: `text-success`
- Muted text: `text-muted-foreground`

---

## Typography

```tsx
// Page titles
<h1 className="text-2xl font-bold tracking-tight">Browse Contracts</h1>

// Section headers
<h2 className="text-lg font-semibold">Active Contracts</h2>

// Body
<p className="text-sm text-muted-foreground">...</p>

// Labels
<span className="text-xs font-medium uppercase tracking-wide text-muted-foreground">...</span>
```

---

## Layout Patterns

### Dashboard Layout Shell

```tsx
export default function DashboardLayout({ children }) {
  return (
    <div className="min-h-screen bg-background">
      <Sidebar />
      <main className="ml-64 p-8">
        <div className="max-w-5xl mx-auto">{children}</div>
      </main>
    </div>
  )
}
```

### Page Header (use on every page)

```tsx
function PageHeader({ title, description, action }: {
  title: string
  description?: string
  action?: React.ReactNode
}) {
  return (
    <div className="flex items-center justify-between mb-6">
      <div>
        <h1 className="text-2xl font-bold tracking-tight">{title}</h1>
        {description && <p className="text-muted-foreground mt-1">{description}</p>}
      </div>
      {action}
    </div>
  )
}
```

---

## Key Component Patterns

### ContractCard (used in browse + dashboard)

```tsx
import { Badge } from '@/components/ui/badge'
import { Card, CardContent, CardFooter, CardHeader } from '@/components/ui/card'
import { formatUSD } from '@/lib/utils/currency'
import type { Contract } from '@/types'

export function ContractCard({ contract }: { contract: Contract }) {
  return (
    <Card className="hover:shadow-md transition-shadow">
      <CardHeader className="pb-2">
        <div className="flex items-start justify-between">
          <h3 className="font-semibold text-base">{contract.title}</h3>
          <StatusBadge status={contract.status} />
        </div>
      </CardHeader>
      <CardContent className="space-y-2 text-sm text-muted-foreground">
        <p>{contract.applications_per_day} applications/day · {contract.duration_days} days</p>
        <p className="line-clamp-2">{contract.description}</p>
        <div className="flex gap-2 flex-wrap">
          {contract.job_boards?.map(b => (
            <Badge key={b} variant="secondary">{b}</Badge>
          ))}
        </div>
      </CardContent>
      <CardFooter className="flex justify-between items-center pt-2 border-t">
        <span className="font-semibold text-brand-600">
          {formatUSD(contract.daily_rate_usd)}/day
        </span>
        <span className="text-xs text-muted-foreground">
          Total: {formatUSD(contract.total_amount_usd)}
        </span>
      </CardFooter>
    </Card>
  )
}
```

### StatusBadge

```tsx
const statusConfig = {
  open:      { label: 'Open',       class: 'bg-green-100 text-green-700' },
  in_review: { label: 'In Review',  class: 'bg-yellow-100 text-yellow-700' },
  active:    { label: 'Active',     class: 'bg-blue-100 text-blue-700' },
  completed: { label: 'Completed',  class: 'bg-gray-100 text-gray-600' },
  cancelled: { label: 'Cancelled',  class: 'bg-red-100 text-red-600' },
  disputed:  { label: 'Disputed',   class: 'bg-orange-100 text-orange-700' },
}

export function StatusBadge({ status }: { status: string }) {
  const config = statusConfig[status] ?? { label: status, class: 'bg-gray-100 text-gray-600' }
  return (
    <span className={`text-xs font-medium px-2 py-1 rounded-full ${config.class}`}>
      {config.label}
    </span>
  )
}
```

### EscrowProgress (on active contract pages)

```tsx
export function EscrowProgress({ released, total }: { released: number, total: number }) {
  const percent = Math.round((released / total) * 100)
  return (
    <div className="space-y-1">
      <div className="flex justify-between text-sm">
        <span className="text-muted-foreground">Escrow Progress</span>
        <span className="font-medium">{formatUSD(released)} / {formatUSD(total)}</span>
      </div>
      <div className="h-2 bg-muted rounded-full overflow-hidden">
        <div
          className="h-full bg-brand-500 rounded-full transition-all"
          style={{ width: `${percent}%` }}
        />
      </div>
      <p className="text-xs text-muted-foreground">{percent}% released</p>
    </div>
  )
}
```

### DailySubmissionCard

```tsx
export function DailySubmissionCard({ submission }: { submission: DailySubmission }) {
  return (
    <Card>
      <CardHeader>
        <div className="flex justify-between items-center">
          <span className="font-medium">
            {new Date(submission.submission_date).toLocaleDateString('en-US', {
              weekday: 'long', month: 'short', day: 'numeric'
            })}
          </span>
          <StatusBadge status={submission.status} />
        </div>
      </CardHeader>
      <CardContent>
        <p className="text-sm text-muted-foreground mb-3">
          {submission.jobs_applied.length} jobs applied
        </p>
        <div className="space-y-2">
          {submission.jobs_applied.map((job, i) => (
            <div key={i} className="flex items-center gap-2 text-sm">
              <span className="font-medium">{job.company}</span>
              <span className="text-muted-foreground">·</span>
              <span>{job.role}</span>
              <a href={job.job_url} target="_blank" className="text-brand-600 hover:underline ml-auto text-xs">
                View
              </a>
            </div>
          ))}
        </div>
      </CardContent>
    </Card>
  )
}
```

---

## Forms

Use `react-hook-form` + `zod` for all forms:

```bash
npm install react-hook-form zod @hookform/resolvers
```

Pattern:
```tsx
'use client'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const schema = z.object({
  title: z.string().min(5, 'Title must be at least 5 characters'),
  daily_rate_usd: z.number().min(1, 'Minimum $1/day'),
  duration_days: z.number().min(7).max(90),
})

type FormData = z.infer<typeof schema>

export function CreateContractForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(schema)
  })

  async function onSubmit(data: FormData) {
    await createContract(data)  // server action
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div>
        <label className="text-sm font-medium">Contract Title</label>
        <Input {...register('title')} placeholder="Apply to 10 SWE jobs per day" />
        {errors.title && <p className="text-xs text-danger mt-1">{errors.title.message}</p>}
      </div>
      {/* ... */}
    </form>
  )
}
```

---

## Responsive Rules

- Mobile-first: design for 375px, enhance for tablet/desktop
- Sidebar collapses to bottom nav on mobile
- Cards go from 1-col (mobile) → 2-col (tablet) → 3-col (desktop) on browse page
- Use `grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4`
- Minimum touch target: 44×44px for all buttons

---

## Empty States

```tsx
function EmptyState({ icon: Icon, title, description, action }: {
  icon: LucideIcon
  title: string
  description: string
  action?: React.ReactNode
}) {
  return (
    <div className="text-center py-16 px-4">
      <div className="mx-auto w-12 h-12 rounded-full bg-muted flex items-center justify-center mb-4">
        <Icon className="w-6 h-6 text-muted-foreground" />
      </div>
      <h3 className="font-semibold text-base mb-1">{title}</h3>
      <p className="text-sm text-muted-foreground mb-4">{description}</p>
      {action}
    </div>
  )
}
```

Usage:
```tsx
<EmptyState
  icon={Briefcase}
  title="No contracts yet"
  description="Post your first contract to find someone to apply to jobs on your behalf."
  action={<Button asChild><Link href="/client/post-contract">Post a Contract</Link></Button>}
/>
```
