---
name: mcp-setup
description: >
  MCP server configuration for Claude Code when working on the JobBridge app.
  Read this when setting up the development environment, configuring Claude Code,
  or when you need to interact with GitHub, Supabase, Stripe, Vercel, or browser
  testing tools. Trigger on: MCP, server config, Claude Code setup, GitHub, Vercel deploy,
  Supabase CLI, or any DevOps/infrastructure task.
---

# MCP Server Setup — JobBridge (Claude Code)

## What Are MCP Servers?

MCP servers let Claude Code directly interact with external tools (GitHub, Supabase, Vercel, etc.)
without you copy-pasting things. Configure them in `~/.claude/settings.json` on your laptop.

---

## Recommended MCP Servers

### 1. GitHub MCP
**What it does:** Create repos, branches, commits, PRs, issues directly from Claude Code.

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "your_token_here"
      }
    }
  }
}
```

**Get token:** github.com → Settings → Developer settings → Personal access tokens → Fine-grained tokens
**Scopes needed:** repo (read/write), pull_requests, issues

**Claude Code can then:**
- Initialize the JobBridge repo
- Create feature branches for each feature
- Open PRs for review
- Manage issues/tasks

---

### 2. Supabase MCP
**What it does:** Run SQL queries, apply migrations, inspect tables, manage RLS policies.

```json
{
  "supabase": {
    "command": "npx",
    "args": ["-y", "@supabase/mcp-server-supabase@latest"],
    "env": {
      "SUPABASE_ACCESS_TOKEN": "your_supabase_personal_access_token"
    }
  }
}
```

**Get token:** supabase.com → Account → Access Tokens

**Claude Code can then:**
- Create tables directly from schema definitions
- Run SQL migrations
- Check RLS policies
- Inspect data during debugging

---

### 3. Vercel MCP
**What it does:** Deploy, check logs, manage environment variables.

```json
{
  "vercel": {
    "command": "npx",
    "args": ["-y", "@vercel/mcp-adapter"],
    "env": {
      "VERCEL_TOKEN": "your_vercel_token"
    }
  }
}
```

**Get token:** vercel.com → Settings → Tokens

**Claude Code can then:**
- Trigger deployments
- Set/update environment variables
- View build logs and errors
- Check deployment status

---

### 4. Browsertools MCP (Puppeteer)
**What it does:** Opens a browser, clicks around, takes screenshots for live UI testing.

```json
{
  "puppeteer": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-puppeteer"]
  }
}
```

**Claude Code can then:**
- Test the app in a real browser
- Verify UI flows (signup, post contract, etc.)
- Catch layout or JS errors

---

### 5. Filesystem MCP (for large codebases)
**What it does:** Gives Claude Code better access to read/write project files.

```json
{
  "filesystem": {
    "command": "npx",
    "args": [
      "-y",
      "@modelcontextprotocol/server-filesystem",
      "/Users/yourname/Projects/jobbridge"
    ]
  }
}
```

Replace the path with your actual project path.

---

## Full `~/.claude/settings.json` Template

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_YOUR_TOKEN"
      }
    },
    "supabase": {
      "command": "npx",
      "args": ["-y", "@supabase/mcp-server-supabase@latest"],
      "env": {
        "SUPABASE_ACCESS_TOKEN": "sbp_YOUR_TOKEN"
      }
    },
    "vercel": {
      "command": "npx",
      "args": ["-y", "@vercel/mcp-adapter"],
      "env": {
        "VERCEL_TOKEN": "your_vercel_token"
      }
    },
    "puppeteer": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-puppeteer"]
    },
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/yourname/Projects/jobbridge"
      ]
    }
  }
}
```

---

## How to Install Skills in Claude Code

Place each SKILL.md in your project under a `.claude/skills/` folder:

```
your-project/
  .claude/
    skills/
      project-overview/SKILL.md
      database-schema/SKILL.md
      nextjs-conventions/SKILL.md
      supabase-patterns/SKILL.md
      payment-integration/SKILL.md
      auth-flows/SKILL.md
      ui-components/SKILL.md
      mcp-setup/SKILL.md
```

Claude Code will automatically read and use these skills.

---

## First-Time Project Setup Checklist

Run these in order when setting up the project from scratch:

```bash
# 1. Create Next.js app
npx create-next-app@latest jobbridge --typescript --tailwind --app --src-dir=false

# 2. Install dependencies
cd jobbridge
npm install @supabase/supabase-js @supabase/auth-helpers-nextjs
npm install stripe @stripe/stripe-js @stripe/react-stripe-js
npm install razorpay
npm install react-hook-form @hookform/resolvers zod
npm install lucide-react
npm install resend
npm install @types/node --save-dev

# 3. Install shadcn/ui
npx shadcn@latest init
npx shadcn@latest add button card badge input textarea select dialog sheet tabs avatar progress separator skeleton toast alert

# 4. Set up Supabase CLI
npm install supabase --save-dev
npx supabase init
npx supabase login

# 5. Link to your Supabase project
npx supabase link --project-ref YOUR_PROJECT_REF

# 6. Create initial migration
npx supabase migration new initial_schema
# Then paste the schema from database-schema skill into the migration file

# 7. Apply migrations
npx supabase db push

# 8. Generate TypeScript types
npx supabase gen types typescript --project-id YOUR_PROJECT_ID > types/supabase.ts
```

---

## Stripe Setup Checklist

1. Create account at stripe.com
2. Get test API keys from Dashboard → Developers → API keys
3. Install Stripe CLI: `brew install stripe/stripe-cli/stripe`
4. Login: `stripe login`
5. Forward webhooks locally: `stripe listen --forward-to localhost:3000/api/webhooks/stripe`
6. Add webhook endpoint in Stripe Dashboard for production (event: `payment_intent.succeeded`)

## Razorpay Setup Checklist

1. Create account at razorpay.com
2. Enable Route (Payouts) in Razorpay Dashboard
3. Complete KYC/business verification
4. Get API keys from Settings → API Keys
5. Add webhook in Dashboard → Webhooks (events: `payout.processed`, `payout.failed`)
6. Fund your Razorpay current account for payouts
