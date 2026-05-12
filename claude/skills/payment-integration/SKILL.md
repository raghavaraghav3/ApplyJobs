---
name: payment-integration
description: >
  Complete Stripe + Razorpay + escrow payment integration for the JobBridge app.
  Always read this before writing any payment, payout, escrow, webhook, Stripe, or
  Razorpay code. Trigger on any mention of: payments, payouts, escrow, Stripe,
  Razorpay, currency, USD, INR, release funds, refund, platform fee, webhook, or
  any money-related logic.
---

# Payment Integration — JobBridge

## Overview

| What | How |
|---|---|
| Client pays upfront (USD) | Stripe PaymentIntent with `capture_method: manual` |
| Funds held in escrow | PaymentIntent captured partially per day |
| Worker receives payout (INR) | Razorpay Payouts API |
| Platform fee | 10% deducted before worker payout |
| Currency conversion | Done by Razorpay at time of payout |

---

## Payment Flow (Step by Step)

```
1. Client hires a worker
2. Platform creates a Stripe PaymentIntent for full contract amount
3. Client pays → funds are authorized but not captured
4. Escrow record created in DB (status: 'funded')
5. Worker submits daily work
6. Client approves (or 24h auto-approve triggers)
7. Platform captures daily_rate_usd from the PaymentIntent (partial capture)
8. Payment record created in DB
9. Platform sends payout to worker via Razorpay
10. Escrow updated (released_amount increases)
11. Contract ends → any remaining uncaptured funds auto-cancelled
```

---

## Stripe Setup

### Install

```bash
npm install stripe @stripe/stripe-js
```

### Create PaymentIntent (when client hires worker)

```ts
// lib/stripe/index.ts
import Stripe from 'stripe'
import { env } from '@/lib/env'

export const stripe = new Stripe(env.stripeSecretKey, {
  apiVersion: '2024-06-20'
})
```

```ts
// Called in a Server Action when client accepts a worker application
export async function createEscrowPaymentIntent(contractId: string, totalAmountUSD: number) {
  const amountInCents = Math.round(totalAmountUSD * 100)

  const paymentIntent = await stripe.paymentIntents.create({
    amount: amountInCents,
    currency: 'usd',
    capture_method: 'manual',       // ← key: authorize but don't capture yet
    metadata: { contractId },
    description: `JobBridge contract ${contractId} escrow`,
  })

  // Save to DB
  const supabase = createServerClient()
  await supabase.from('escrow_accounts').insert({
    contract_id: contractId,
    total_amount_usd: totalAmountUSD,
    stripe_payment_intent_id: paymentIntent.id,
    status: 'pending'
  })

  return paymentIntent.client_secret  // send to client to confirm payment
}
```

### Client-side Payment Confirmation

```tsx
// components/client/PaymentForm.tsx
'use client'
import { loadStripe } from '@stripe/stripe-js'
import { Elements, PaymentElement, useStripe, useElements } from '@stripe/react-stripe-js'

const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY!)

export function PaymentForm({ clientSecret }: { clientSecret: string }) {
  return (
    <Elements stripe={stripePromise} options={{ clientSecret }}>
      <CheckoutForm />
    </Elements>
  )
}

function CheckoutForm() {
  const stripe = useStripe()
  const elements = useElements()

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    if (!stripe || !elements) return

    const { error } = await stripe.confirmPayment({
      elements,
      confirmParams: { return_url: `${window.location.origin}/client/dashboard` }
    })
    if (error) console.error(error.message)
  }

  return (
    <form onSubmit={handleSubmit}>
      <PaymentElement />
      <button type="submit">Fund Contract</button>
    </form>
  )
}
```

### Partial Capture (Daily Payment Release)

```ts
// Called when client approves a daily submission
export async function releaseDailyPayment(submissionId: string) {
  const supabase = createServerClient()

  // Get submission + contract + escrow
  const { data: submission } = await supabase
    .from('daily_submissions')
    .select('*, contracts(*, escrow_accounts(*))')
    .eq('id', submissionId)
    .single()

  if (!submission || submission.status !== 'approved') throw new Error('Invalid submission')

  const contract = submission.contracts
  const escrow = contract.escrow_accounts[0]
  const dailyRate = contract.daily_rate_usd
  const platformFee = calculatePlatformFee(dailyRate)
  const netPayout = calculateNetPayout(dailyRate)

  // Capture partial amount from Stripe
  await stripe.paymentIntents.capture(escrow.stripe_payment_intent_id, {
    amount_to_capture: Math.round(dailyRate * 100)
  })

  // Create payment record
  await supabase.from('payments').insert({
    contract_id: contract.id,
    submission_id: submissionId,
    worker_id: contract.worker_id,
    client_id: contract.client_id,
    gross_amount_usd: dailyRate,
    platform_fee_usd: platformFee,
    net_amount_usd: netPayout,
    status: 'processing'
  })

  // Update escrow
  await supabase.from('escrow_accounts')
    .update({ released_amount_usd: escrow.released_amount_usd + dailyRate })
    .eq('id', escrow.id)

  // Trigger Razorpay payout
  await initiateRazorpayPayout(contract.worker_id, netPayout, contract.id)
}
```

### Stripe Webhook Handler

```ts
// app/api/webhooks/stripe/route.ts
import { stripe } from '@/lib/stripe'
import { headers } from 'next/headers'

export async function POST(req: Request) {
  const body = await req.text()
  const sig = headers().get('stripe-signature')!

  let event: Stripe.Event
  try {
    event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!)
  } catch {
    return new Response('Webhook signature failed', { status: 400 })
  }

  switch (event.type) {
    case 'payment_intent.succeeded':
      // Mark escrow as funded
      const pi = event.data.object as Stripe.PaymentIntent
      const supabase = createServerClient()
      await supabase.from('escrow_accounts')
        .update({ status: 'funded', funded_at: new Date().toISOString() })
        .eq('stripe_payment_intent_id', pi.id)
      break
    // handle other events as needed
  }

  return new Response('ok')
}
```

---

## Razorpay Setup (Worker Payouts in INR)

### Install

```bash
npm install razorpay
```

### Initialize

```ts
// lib/razorpay/index.ts
import Razorpay from 'razorpay'
import { env } from '@/lib/env'

export const razorpay = new Razorpay({
  key_id: env.razorpayKeyId,
  key_secret: env.razorpayKeySecret
})
```

### Register Worker for Payouts (one-time during onboarding)

```ts
export async function registerWorkerForPayouts(workerId: string, workerData: {
  name: string
  email: string
  phone: string
  bankAccount: string
  ifscCode: string
}) {
  const supabase = createServerClient()

  // Create Razorpay Contact
  const contact = await razorpay.contacts.create({
    name: workerData.name,
    email: workerData.email,
    contact: workerData.phone,
    type: 'employee',
    reference_id: workerId
  })

  // Create Fund Account (bank account)
  const fundAccount = await razorpay.fundAccount.create({
    contact_id: contact.id,
    account_type: 'bank_account',
    bank_account: {
      name: workerData.name,
      ifsc: workerData.ifscCode,
      account_number: workerData.bankAccount
    }
  })

  // Save IDs to worker profile
  await supabase.from('worker_profiles').update({
    razorpay_contact_id: contact.id,
    razorpay_fund_account_id: fundAccount.id
  }).eq('id', workerId)
}
```

### Initiate Payout to Worker

```ts
export async function initiateRazorpayPayout(workerId: string, amountUSD: number, contractId: string) {
  const supabase = createServerClient()

  // Get worker's Razorpay fund account
  const { data: workerProfile } = await supabase
    .from('worker_profiles')
    .select('razorpay_fund_account_id')
    .eq('id', workerId)
    .single()

  if (!workerProfile?.razorpay_fund_account_id) throw new Error('Worker payout not configured')

  // Convert USD to INR (use live rate via Razorpay or a rates API)
  const usdToInr = await getUsdToInrRate()
  const amountINR = Math.round(amountUSD * usdToInr * 100) // paise

  const payout = await razorpay.payouts.create({
    account_number: process.env.RAZORPAY_ACCOUNT_NUMBER!,  // platform's Razorpay account
    fund_account_id: workerProfile.razorpay_fund_account_id,
    amount: amountINR,
    currency: 'INR',
    mode: 'IMPS',
    purpose: 'payout',
    queue_if_low_balance: true,
    reference_id: `${contractId}-${Date.now()}`,
    narration: 'JobBridge earnings'
  })

  // Update payment record
  await supabase.from('payments')
    .update({
      razorpay_payout_id: payout.id,
      inr_equivalent: amountINR / 100,
      status: payout.status === 'processed' ? 'completed' : 'processing'
    })
    .eq('contract_id', contractId)
    .eq('status', 'processing')
    .order('created_at', { ascending: false })
    .limit(1)
}
```

### Get Live USD to INR Rate

```ts
export async function getUsdToInrRate(): Promise<number> {
  try {
    const res = await fetch('https://api.exchangerate-api.com/v4/latest/USD')
    const data = await res.json()
    return data.rates.INR ?? 83  // fallback to approximate
  } catch {
    return 83  // safe fallback
  }
}
```

### Razorpay Webhook Handler

```ts
// app/api/webhooks/razorpay/route.ts
import crypto from 'crypto'

export async function POST(req: Request) {
  const body = await req.text()
  const sig = req.headers.get('x-razorpay-signature')!
  const secret = process.env.RAZORPAY_WEBHOOK_SECRET!

  const expected = crypto.createHmac('sha256', secret).update(body).digest('hex')
  if (sig !== expected) return new Response('Invalid signature', { status: 400 })

  const event = JSON.parse(body)

  if (event.event === 'payout.processed') {
    const supabase = createServerClient()
    await supabase.from('payments')
      .update({ status: 'completed' })
      .eq('razorpay_payout_id', event.payload.payout.entity.id)
  }

  if (event.event === 'payout.failed') {
    const supabase = createServerClient()
    await supabase.from('payments')
      .update({ status: 'failed' })
      .eq('razorpay_payout_id', event.payload.payout.entity.id)
    // TODO: alert admin, retry logic
  }

  return new Response('ok')
}
```

---

## Platform Fee Summary

- Platform takes **10%** of each daily payment
- Example: Daily rate $2 → Fee $0.20 → Worker receives $1.80 (≈ ₹149 at ₹83/USD)
- Fee is deducted before Razorpay payout
- Stripe fees (≈ 2.9% + 30¢) are absorbed by the platform or added to client's price

---

## Refund Logic

If a contract is cancelled before start or mid-way:

```ts
export async function refundEscrow(contractId: string) {
  const supabase = createServerClient()
  const { data: escrow } = await supabase
    .from('escrow_accounts')
    .select('*')
    .eq('contract_id', contractId)
    .single()

  // Cancel the PaymentIntent (releases uncaptured funds)
  await stripe.paymentIntents.cancel(escrow.stripe_payment_intent_id)

  await supabase.from('escrow_accounts')
    .update({ status: 'refunded' })
    .eq('id', escrow.id)
}
```
