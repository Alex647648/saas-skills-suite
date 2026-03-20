# SaaS Skills Suite

A battle-tested collection of 9 Claude Code skills for building SaaS products with **Next.js + Supabase + Stripe**.

Extracted from real production development (InspirationLab Online), covering the full lifecycle from project scaffolding to deployment.

## Skills Overview

```
project-scaffold          # Architecture docs + constraint code skeleton
    |
saas-quickstart           # 5-step launch from starter kit
    |
    +-- nextjs-fullstack  # Next.js App Router patterns
    +-- supabase-developer # Supabase full-stack reference
    +-- stripe-payments    # Stripe integration (beginner)
    |
mvp-billing-system        # Production billing system (EN)
mvp-billing-system-cn     # Production billing system (CN)
    |
supabase-gemini-deploy    # Edge Function deployment troubleshooting
    |
deploy-gate               # Pre-deployment 6-gate checks
```

## Skill Details

| Skill | Lines | Description |
|-------|-------|-------------|
| `project-scaffold` | ~140 | AI-native project skeleton generator. Generates 8 coupled architecture docs + TypeScript constraint code |
| `saas-quickstart` | ~70 | Quick start from `next-supabase-stripe-starter` kit |
| `nextjs-fullstack` | ~140 | Next.js App Router (14/15+): Server Components, Server Actions, Middleware |
| `supabase-developer` | ~1470 | Supabase full reference: Auth (`@supabase/ssr`), Database, Storage, Real-time, Edge Functions |
| `stripe-payments` | ~200 | Stripe integration beginner guide: Checkout, Webhooks, Subscription gating |
| `mvp-billing-system` | ~1300 | Production billing system (EN): dual payment, dual wallet, 8 pitfalls, testing, monitoring |
| `mvp-billing-system-cn` | ~1300 | Same as above, in Chinese |
| `supabase-gemini-deploy` | ~520 | Edge Function deployment troubleshooting: JWT (ES256/HS256), CORS, rate limiting |
| `deploy-gate` | ~100 | Pre-deployment gate: build, constraints, env vars, regression, git, final check |

## Installation

Copy the skills you need into your Claude Code skills directory:

```bash
# Clone the repo
git clone https://github.com/Alex647648/saas-skills-suite.git

# Copy all skills
cp -r saas-skills-suite/*/ ~/.claude/skills/

# Or copy specific skills
cp -r saas-skills-suite/mvp-billing-system ~/.claude/skills/
cp -r saas-skills-suite/deploy-gate ~/.claude/skills/
```

## Tech Stack

These skills are designed for:

- **Language**: TypeScript
- **Frontend**: Next.js 14/15+ (App Router)
- **Backend**: Supabase Edge Functions (Deno) + Next.js API Routes
- **Database**: PostgreSQL (Supabase)
- **Auth**: Supabase Auth (`@supabase/ssr`)
- **Payments**: Stripe (Checkout, Subscriptions, Webhooks)
- **Deployment**: Vercel + Supabase

## Key Features Covered

- Dual payment (Stripe auto-renew + manual subscription)
- Dual wallet (subscription credits + top-up credits)
- Credit-based billing with atomic RPC deduction
- Webhook idempotency (event dedup table + composite keys)
- Stripe API version compatibility (`extractPeriod()` helper)
- Edge Function JWT troubleshooting (ES256 vs HS256)
- DB-backed rate limiting (survives cold starts)
- 3-layer downgrade prevention
- Structured logging + monitoring queries
- Pre-deployment gate checks

## License

MIT
