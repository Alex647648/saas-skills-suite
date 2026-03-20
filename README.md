# SaaS Skills Suite

**[中文版 README](./README_CN.md)**

> 9 production-tested [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills that guide AI through every stage of SaaS development — from project scaffolding to deployment gate checks.

Built on real lessons from shipping a SaaS product (Next.js + Supabase + Stripe) over 4 days and 32 commits. Every pitfall section is a bug we actually hit in production.

---

## What Are Claude Code Skills?

Skills are markdown instruction files that teach Claude Code *how* to do specific tasks. When you mention a keyword like "billing system" or "deploy", Claude Code automatically loads the matching skill and follows its patterns, code templates, and best practices.

Think of them as **reusable expert knowledge** that turns Claude Code into a specialist.

---

## Who Is This For?

- **Solo developers** building SaaS products with Next.js + Supabase + Stripe
- **Small teams** who want consistent architecture decisions across AI-assisted development
- **Anyone** who wants to skip the "figure it out" phase of SaaS billing, auth, and deployment

---

## Quick Start

### Option 1: Install All Skills (Recommended)

```bash
git clone https://github.com/Alex647648/saas-skills-suite.git
cp -r saas-skills-suite/*/ ~/.claude/skills/
```

### Option 2: Install Specific Skills

```bash
git clone https://github.com/Alex647648/saas-skills-suite.git

# Example: only billing + deployment
cp -r saas-skills-suite/mvp-billing-system ~/.claude/skills/
cp -r saas-skills-suite/deploy-gate ~/.claude/skills/
```

### Verify Installation

Open Claude Code and type:

```
/skills
```

You should see the installed skills listed. Try saying "help me set up Stripe payments" — the relevant skill activates automatically.

---

## Skills at a Glance

| # | Skill | Lines | What It Does |
|---|-------|------:|-------------|
| 1 | [project-scaffold](#1-project-scaffold) | 139 | Generates 8 architecture docs + constraint code skeleton |
| 2 | [saas-quickstart](#2-saas-quickstart) | 71 | 5-step launch guide from starter kit to production |
| 3 | [nextjs-fullstack](#3-nextjs-fullstack) | 141 | Next.js App Router patterns (Server Components, Actions, Middleware) |
| 4 | [supabase-developer](#4-supabase-developer) | 1,492 | Supabase full reference (Auth, DB, Storage, Real-time, Edge Functions) |
| 5 | [stripe-payments](#5-stripe-payments) | 224 | Stripe integration beginner guide (Checkout, Webhooks, Gating) |
| 6 | [mvp-billing-system](#6-mvp-billing-system) | 1,342 | Production billing system — the full playbook (English) |
| 7 | [mvp-billing-system-cn](#7-mvp-billing-system-cn) | 1,335 | Same as above, in Chinese |
| 8 | [supabase-gemini-deploy](#8-supabase-gemini-deploy) | 522 | Edge Function deployment troubleshooting |
| 9 | [deploy-gate](#9-deploy-gate) | 100 | Pre-deployment 6-gate quality checks |

**Total: ~5,366 lines of structured knowledge + 771 lines of templates**

---

## How the Skills Connect

```
                    +-----------------------+
                    |   project-scaffold    |  "Start a new project"
                    |  (Architecture docs   |
                    |   + code skeleton)    |
                    +-----------+-----------+
                                |
                    +-----------v-----------+
                    |    saas-quickstart    |  "Quick launch from starter kit"
                    +-----------+-----------+
                                |
              +-----------------+-----------------+
              |                 |                 |
    +---------v-------+ +------v--------+ +------v----------+
    | nextjs-fullstack| | supabase-     | | stripe-payments |
    | (App Router,    | | developer     | | (Beginner:      |
    |  SSR, Actions)  | | (Auth, DB,    | |  Checkout,      |
    |                 | |  Storage, RT) | |  Webhooks)      |
    +--------+--------+ +------+--------+ +------+----------+
              |                 |                 |
              +-----------------+-----------------+
                                |
              +-----------------+-----------------+
              |                                   |
    +---------v-----------+          +------------v-----------+
    | mvp-billing-system  |          | supabase-gemini-deploy |
    | (Production-grade:  |          | (Edge Function         |
    |  dual wallet, dual  |          |  troubleshooting:      |
    |  payment, 8 pitfalls|          |  JWT, CORS, 401/500)   |
    |  testing, monitoring)|         +------------+-----------+
    +---------+-----------+                       |
              |                                   |
              +-----------------------------------+
                                |
                    +-----------v-----------+
                    |      deploy-gate      |  "Can I deploy?"
                    |  (6-gate pre-deploy   |
                    |   quality checks)     |
                    +-----------------------+
```

Every skill has a **Related Skills** table at the bottom linking to its neighbors, so Claude Code always knows where to go next.

---

## Detailed Skill Descriptions

### 1. project-scaffold

> **Trigger keywords**: "new project", "scaffold", "initialize", "architecture", "from scratch"

A 4-phase workflow that generates a complete project skeleton:

1. **Collect** — Interactive Q&A to gather product requirements
2. **Generate 8 Docs** — Validation targets, data model, API contract, design system, agent spec, dev constraints, task board, deploy checklist
3. **Generate Constraint Code** — Branded types, state machines, confirm guards, ESLint rules, lint scripts
4. **Validate** — Run consistency checks across all generated files

Includes template files in `references/templates/` for document and code generation.

---

### 2. saas-quickstart

> **Trigger keywords**: "new SaaS", "create project", "quick launch", "SaaS template"

The fastest path from zero to deployed SaaS:

| Step | What | Time |
|------|------|------|
| 1 | Clone starter kit | 10 min |
| 2 | Configure Supabase + Stripe | 30 min |
| 3 | Add your tool/feature | 1-3 days |
| 4 | Add payment gating | 30 min |
| 5 | Deploy to Vercel | 10 min |

---

### 3. nextjs-fullstack

> **Trigger keywords**: "Next.js", "page", "route", "Server Action"

Covers Next.js 14/15+ App Router patterns:

- Server Components vs Client Components (when to use `'use client'`)
- Server Actions for mutations (with Zod validation)
- Route Handlers for webhooks
- Middleware for auth protection
- Performance targets (TTFB < 200ms, LCP < 2.5s)

---

### 4. supabase-developer

> **Trigger keywords**: "Supabase", "database", "auth", "real-time", "storage", "Edge Function"

The most comprehensive skill (1,492 lines). Covers:

- **Auth**: `@supabase/ssr` setup (updated from deprecated `auth-helpers`), `getUser()` vs `getSession()`, OAuth, middleware
- **Database**: Schema design, RLS policies, migrations, RPC functions, `FOR UPDATE` locking
- **Storage**: Buckets, policies, signed URLs, image transforms
- **Real-time**: Channels, presence, broadcast
- **Edge Functions**: Deno runtime, CORS, shared modules, deployment

---

### 5. stripe-payments

> **Trigger keywords**: "Stripe beginner", "payment intro", "simple subscription"

Beginner-level Stripe integration using Next.js Route Handlers:

- Checkout Session creation
- Webhook event handling
- Subscription status sync to database
- Payment gating in pages

**Note**: This is the entry-level skill. For production billing with credits, dual wallets, and idempotency, use `mvp-billing-system` instead.

---

### 6. mvp-billing-system

> **Trigger keywords**: "billing system", "Stripe integration", "payment system", "credits", "subscription"

The crown jewel (1,342 lines). A complete playbook for production SaaS billing:

| Phase | Content |
|-------|---------|
| Phase 1 | Database schema (5 core tables, RLS, triggers, indexes) |
| Phase 2 | Billing RPC functions (atomic deduction, idempotent grants) |
| Phase 3 | Edge Functions — Stripe integration (checkout, webhook, management) |
| Phase 4 | Frontend components (pricing panel, paywall, billing history) |
| Phase 5 | Advanced — dual payment + dual wallet |
| Phase 6 | Security hardening (rate limiting, audit logs, pg_cron) |

Plus:
- **8 Pitfall Sections** — Real bugs we hit and how we fixed them (Stripe API version hell, JWT ES256/HS256, duplicate subscriptions, etc.)
- **Idempotency Patterns** — Webhook dedup, credit grant composite keys, refund checks
- **Testing & QA** — Stripe CLI webhook forwarding, Test Clocks, test scenarios checklist
- **Monitoring** — Structured logging, health check SQL queries, alerting strategy
- **Debugging Quick Reference** — Symptom → Cause → Fix table

---

### 7. mvp-billing-system-cn

Same content as `mvp-billing-system`, written entirely in Chinese. Choose based on your preference.

---

### 8. supabase-gemini-deploy

> **Trigger keywords**: "Edge Function", "deploy error", "401", "500", "CORS", "JWT"

A troubleshooting manual for Supabase Edge Function deployment:

- JWT authentication issues (ES256 vs HS256 mismatch)
- CORS configuration
- API Key and environment variable setup
- Billing RPC debugging
- Rate limiting configuration
- Network diagnostics (VPN/TLS issues)

Organized as a decision tree: symptom → diagnosis → fix.

---

### 9. deploy-gate

> **Trigger keywords**: "deploy", "release", "go live", "can I ship"

6 sequential gates — all must pass before deployment:

| Gate | Check | Auto? |
|------|-------|-------|
| 1 | Build + TypeScript + ESLint | Yes |
| 2 | Constraint scripts (if exist) | Yes |
| 3 | Environment variables + migrations | Yes |
| 4 | Feature regression checklist | Manual |
| 5 | Git status + branch + changelog | Yes |
| 6 | Final summary report | — |

Gate 2 gracefully skips if constraint scripts don't exist (they're generated by `project-scaffold`).

---

## Tech Stack

These skills are designed for the following stack, but many patterns are transferable:

| Layer | Technology |
|-------|-----------|
| Language | TypeScript |
| Frontend | Next.js 14/15+ (App Router) |
| Backend | Supabase Edge Functions (Deno) + Next.js API Routes |
| Database | PostgreSQL (Supabase) |
| Auth | Supabase Auth (`@supabase/ssr`) |
| Payments | Stripe (Checkout, Subscriptions, Webhooks) |
| Deployment | Vercel + Supabase |

---

## Customization

Each skill is a standalone markdown file. To customize:

1. **Edit the SKILL.md** — Change code templates, add your own patterns
2. **Adjust trigger keywords** — Modify the `description` field in the YAML frontmatter
3. **Add project-specific context** — Add your price IDs, table names, API endpoints
4. **Remove what you don't need** — Each skill works independently

### File Structure

```
~/.claude/skills/
├── project-scaffold/
│   ├── SKILL.md
│   └── references/templates/    # Doc + code templates
├── saas-quickstart/
│   └── SKILL.md
├── nextjs-fullstack/
│   └── SKILL.md
├── supabase-developer/
│   └── SKILL.md
├── stripe-payments/
│   └── SKILL.md
├── mvp-billing-system/
│   └── SKILL.md
├── mvp-billing-system-cn/
│   └── SKILL.md
├── supabase-gemini-deploy/
│   └── SKILL.md
└── deploy-gate/
    └── SKILL.md
```

---

## FAQ

**Q: Do I need all 9 skills?**
No. Each skill works independently. Start with the ones relevant to your current task. The `mvp-billing-system` + `deploy-gate` combo covers the most ground.

**Q: Will these conflict with other Claude Code skills?**
No. Skills are loaded on-demand based on keyword matching. They don't interfere with each other or with skills from other sources.

**Q: Can I use these with a different tech stack?**
The architectural patterns (idempotency, webhook handling, credit systems) are stack-agnostic. The code templates are specific to Next.js + Supabase + Stripe, but can be adapted.

**Q: How do I update the skills?**
Pull the latest changes and re-copy:
```bash
cd saas-skills-suite && git pull
cp -r */* ~/.claude/skills/
```

---

## License

MIT
