<div align="center">

# SaaS Skills Suite

**Production-tested Claude Code skills for building SaaS products from scratch**

[![GitHub Stars](https://img.shields.io/github/stars/Alex647648/saas-skills-suite?style=flat-square)](https://github.com/Alex647648/saas-skills-suite/stargazers)
[![GitHub Forks](https://img.shields.io/github/forks/Alex647648/saas-skills-suite?style=flat-square)](https://github.com/Alex647648/saas-skills-suite/network)
[![GitHub Issues](https://img.shields.io/github/issues/Alex647648/saas-skills-suite?style=flat-square)](https://github.com/Alex647648/saas-skills-suite/issues)
[![GitHub Pull Requests](https://img.shields.io/github/issues-pr/Alex647648/saas-skills-suite?style=flat-square)](https://github.com/Alex647648/saas-skills-suite/pulls)

[![GitHub License](https://img.shields.io/github/license/Alex647648/saas-skills-suite?style=flat-square)](https://github.com/Alex647648/saas-skills-suite/blob/main/LICENSE)
[![Version](https://img.shields.io/badge/version-v1.0.0-green.svg?style=flat-square)](https://github.com/Alex647648/saas-skills-suite)
[![Skills](https://img.shields.io/badge/skills-10-blue.svg?style=flat-square)](https://github.com/Alex647648/saas-skills-suite)
[![Lines](https://img.shields.io/badge/knowledge-5%2C772_lines-orange.svg?style=flat-square)](https://github.com/Alex647648/saas-skills-suite)

[English](./README.md) | [中文文档](./README_CN.md)

</div>

> [!NOTE]
> **What are Claude Code Skills?** Skills are markdown instruction files placed in `~/.claude/skills/` that teach [Claude Code](https://docs.anthropic.com/en/docs/claude-code) *how* to do specific tasks. When you mention a keyword like "billing system" or "deploy", Claude Code automatically loads the matching skill and follows its patterns, code templates, and best practices. Think of them as **reusable expert knowledge** that turns Claude Code into a domain specialist.

## ⚡ Overview

**SaaS Skills Suite** is a collection of 10 Claude Code skills covering the **full SaaS development lifecycle** plus documentation standards — from project scaffolding to production deployment gate checks.

Every skill was extracted from a real production project ([InspirationLab Online](https://www.inspirationlab.net)) built with Next.js + Supabase + Stripe over 4 days and 32 commits. Every pitfall section is a bug we actually hit. Every code template is copy-paste ready.

Compared to generic AI coding assistance, this suite offers 🚀 **6 key advantages**:

1. **Battle-Tested, Not Theoretical**: Every code template, every pitfall, every debugging tip comes from shipping a real product. Not a tutorial — a war diary.

2. **Full Lifecycle Coverage**: From `project-scaffold` (Day 0 architecture) through `mvp-billing-system` (Day 2-3 billing) to `deploy-gate` (Day 4 ship) — no gaps between skills.

3. **Deep Billing Expertise**: 1,300+ lines dedicated to the hardest part of SaaS — payments. Dual wallets, dual payment rails, credit idempotency, Stripe API version hell, webhook dedup. Problems that take weeks to debug, documented in minutes.

4. **Interconnected Skill Graph**: Every skill has a "Related Skills" table. Claude Code always knows which skill to load next. No dead ends, no orphan knowledge.

5. **Bilingual (EN + CN)**: The core billing skill ships in both English and Chinese. Choose the language that matches your thinking.

6. **Zero Lock-In**: Pure markdown files. Edit them, fork them, replace code templates with your own. Works with any Claude Code setup, no plugins or extensions needed.

> These skills are designed for **Next.js + Supabase + Stripe**, but the architectural patterns (idempotency, webhook handling, credit systems, deployment gates) transfer to any stack.

## 📦 Quick Start

### Install All Skills (Recommended)

```bash
git clone https://github.com/Alex647648/saas-skills-suite.git
cp -r saas-skills-suite/*/ ~/.claude/skills/
```

### Install Specific Skills

```bash
git clone https://github.com/Alex647648/saas-skills-suite.git

# Example: only billing + deployment
cp -r saas-skills-suite/mvp-billing-system ~/.claude/skills/
cp -r saas-skills-suite/deploy-gate ~/.claude/skills/
```

### Verify Installation

```bash
# Open Claude Code and type:
/skills
# You should see the installed skills listed

# Try saying:
"help me set up Stripe payments"
# → stripe-payments skill activates automatically
```

## 🏗️ Architecture

### Skill Dependency Graph

The 9 skills form a layered architecture. Upper layers depend on lower layers for context:

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│    project-scaffold              "Start a new project"  │
│    ┌──────────────────────────┐                         │
│    │ 8 Architecture Docs      │                         │
│    │ + Constraint Code        │                         │
│    └────────────┬─────────────┘                         │
│                 │                                       │
│    saas-quickstart               "Quick launch"         │
│    ┌────────────┴─────────────┐                         │
│    │ 5-Step Starter Kit Guide │                         │
│    └────────────┬─────────────┘                         │
│                 │                                       │
│    ┌────────────┼─────────────────────┐                 │
│    │            │                     │                 │
│    ▼            ▼                     ▼                 │
│  nextjs-     supabase-          stripe-                 │
│  fullstack   developer          payments                │
│  (141 lines) (1,492 lines)      (224 lines)             │
│    │            │                     │                 │
│    └────────────┼─────────────────────┘                 │
│                 │                                       │
│    ┌────────────┴─────────────────────┐                 │
│    │                                  │                 │
│    ▼                                  ▼                 │
│  mvp-billing-system            supabase-gemini-         │
│  (1,342 lines EN)              deploy                   │
│  (1,335 lines CN)              (522 lines)              │
│    │                                  │                 │
│    └────────────┬─────────────────────┘                 │
│                 │                                       │
│                 ▼                                       │
│           deploy-gate                "Can I ship?"      │
│           (100 lines)                                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Skill Details

| # | Skill | Lines | Triggers On | What It Does |
|:-:|-------|------:|-------------|-------------|
| 1 | **project-scaffold** | 139 | "new project", "scaffold", "initialize" | Generates 8 coupled architecture docs + TypeScript constraint code skeleton |
| 2 | **saas-quickstart** | 71 | "new SaaS", "quick launch", "starter kit" | 5-step guide from clone to deploy using `next-supabase-stripe-starter` |
| 3 | **nextjs-fullstack** | 141 | "Next.js", "route", "Server Action" | App Router patterns: Server Components, Actions, Middleware, caching |
| 4 | **supabase-developer** | 1,492 | "Supabase", "database", "auth", "Edge Function" | Full reference: Auth (`@supabase/ssr`), DB, Storage, Real-time, Edge Functions |
| 5 | **stripe-payments** | 224 | "Stripe beginner", "payment intro" | Beginner Stripe: Checkout, Webhooks, subscription gating |
| 6 | **mvp-billing-system** | 1,342 | "billing system", "credits", "subscription" | Production billing playbook (EN): dual wallet, 8 pitfalls, testing, monitoring |
| 7 | **mvp-billing-system-cn** | 1,335 | "计费系统", "支付", "订阅" | Same as above, entirely in Chinese |
| 8 | **supabase-gemini-deploy** | 522 | "Edge Function error", "401", "CORS" | Deployment troubleshooting: JWT (ES256/HS256), CORS, rate limiting |
| 9 | **deploy-gate** | 100 | "deploy", "release", "go live" | 6 sequential quality gates before production deployment |
| 10 | **readme-standard** | 406 | "write README", "update README", "project docs" | Standardized README writing guide with badges, structure, bilingual templates |

## 🔍 Skill Deep Dive

### 1. project-scaffold

> **4-phase workflow** that generates a complete project skeleton

| Phase | Action | Output |
|:-----:|--------|--------|
| 1 | Interactive Q&A to gather requirements | Project brief |
| 2 | Generate 8 architecture docs | Validation targets, data model, API contract, design system, agent spec, dev constraints, task board, deploy checklist |
| 3 | Generate constraint code | Branded types, state machines, confirm guards, ESLint rules, lint scripts |
| 4 | Validate consistency | Cross-doc consistency checks pass |

Includes template files in `references/templates/` for document and code generation.

---

### 2. saas-quickstart

> **Fastest path** from zero to deployed SaaS

| Step | What | Time |
|:----:|------|:----:|
| 1 | Clone starter kit | 10 min |
| 2 | Configure Supabase + Stripe | 30 min |
| 3 | Build your core feature | 1-3 days |
| 4 | Add payment gating | 30 min |
| 5 | Deploy to Vercel | 10 min |

---

### 3. nextjs-fullstack

> **Next.js 14/15+ App Router** patterns reference

- Server Components vs Client Components — when to use `'use client'`
- Server Actions for mutations with Zod validation
- Route Handlers for external webhooks
- Middleware for auth protection
- Performance targets: TTFB < 200ms, LCP < 2.5s

---

### 4. supabase-developer

> **The most comprehensive skill** — 1,492 lines covering every Supabase feature

| Module | Key Topics |
|--------|-----------|
| **Auth** | `@supabase/ssr` setup, `getUser()` vs `getSession()`, OAuth, middleware (updated from deprecated `auth-helpers`) |
| **Database** | Schema design, RLS policies, migrations, RPC functions, `FOR UPDATE` locking |
| **Storage** | Buckets, policies, signed URLs, image transforms |
| **Real-time** | Channels, presence, broadcast |
| **Edge Functions** | Deno runtime, CORS, shared modules, `--no-verify-jwt` deployment |

---

### 5. stripe-payments

> **Beginner-level** Stripe integration using Next.js Route Handlers

Covers Checkout Session creation, webhook handling, subscription sync, and payment gating.

> **Note**: For production billing with credits, dual wallets, and idempotency, use **mvp-billing-system** instead.

---

### 6. mvp-billing-system

> **The crown jewel** — 1,342 lines. A complete production billing playbook.

| Phase | Content |
|:-----:|---------|
| 1 | Database schema — 5 core tables, RLS, triggers, indexes |
| 2 | Billing RPC functions — atomic deduction with `FOR UPDATE`, idempotent grants |
| 3 | Edge Functions — Stripe checkout (3 flows), webhook handler, subscription management |
| 4 | Frontend components — pricing panel, paywall modal, billing history |
| 5 | Advanced — dual payment (auto-renew + manual) + dual wallet (subscription + top-up) |
| 6 | Security — DB rate limiting, audit logs, pg_cron cleanup |

**Bonus sections that save you weeks**:

| Section | What You Get |
|---------|-------------|
| 8 Pitfalls | Stripe API version hell, JWT ES256/HS256, duplicate subscriptions, webhook version mismatch, and more |
| Idempotency Patterns | Webhook dedup table, credit grant composite keys, refund checks, top-up dedup |
| Testing & QA | Stripe CLI webhook forwarding, Test Clocks, 20+ test scenarios checklist |
| Monitoring | Structured JSON logging, 5 health-check SQL queries, alerting strategy table |
| Debugging | Symptom → Cause → Fix quick reference (10 common issues) |

---

### 7. mvp-billing-system-cn

Same content as `mvp-billing-system`, written entirely in Chinese. Choose based on your preference.

---

### 8. supabase-gemini-deploy

> **Troubleshooting manual** for Supabase Edge Function deployment

Organized as a decision tree — symptom → diagnosis → fix:

- JWT issues (ES256 vs HS256 mismatch)
- CORS configuration
- API Key and environment variable setup
- Billing RPC debugging
- Rate limiting
- Network diagnostics (VPN/TLS)

---

### 9. deploy-gate

> **6 sequential gates** — all must pass before deployment

| Gate | Check | Auto |
|:----:|-------|:----:|
| 1 | Build + TypeScript + ESLint | Yes |
| 2 | Constraint scripts (graceful skip if absent) | Yes |
| 3 | Environment variables + migration files | Yes |
| 4 | Feature regression checklist | Manual |
| 5 | Git status + branch + changelog | Yes |
| 6 | Final summary report | — |

---

### 10. readme-standard

> **Standardized GitHub README writing guide** — the skill that wrote this README

A complete template system for professional project documentation:

- **9-section standard structure**: Header, Note, Overview, Quick Start, Architecture, Deep Dive, Tech Stack, Customization, FAQ, License
- **Shields.io badge library**: Community metrics, project metadata, tech stack, status badges — all `flat-square` style
- **Emoji section header reference**: 13 standard emoji-to-section mappings
- **Architecture diagram rules**: Box-drawing characters, 70-column width limit, data flow annotations
- **Bilingual spec**: File naming, structure alignment, spacing rules for EN+CN
- **Anti-patterns**: 9 common README mistakes and how to avoid them

## ⚙️ Tech Stack

These skills are built for the following stack. Many patterns are stack-agnostic.

| Layer | Technology |
|-------|-----------|
| Language | TypeScript |
| Frontend | Next.js 14/15+ (App Router) |
| Backend | Supabase Edge Functions (Deno) + Next.js API Routes |
| Database | PostgreSQL (Supabase) |
| Auth | Supabase Auth (`@supabase/ssr`) |
| Payments | Stripe (Checkout, Subscriptions, Webhooks) |
| Deployment | Vercel + Supabase |

## 🔧 Customization

Each skill is a standalone markdown file. To customize:

1. **Edit `SKILL.md`** — Modify code templates, add your own patterns
2. **Adjust triggers** — Change the `description` field in the YAML frontmatter
3. **Add project context** — Insert your price IDs, table names, API endpoints
4. **Remove what you don't need** — Each skill works independently

### File Structure

```
~/.claude/skills/
├── project-scaffold/
│   ├── SKILL.md                             # Skeleton generator skill
│   └── references/templates/                # Doc + code templates
│       ├── constraint-code.md               # TypeScript constraint templates
│       ├── doc-templates.md                 # 8 architecture doc templates
│       └── validation-target.md             # Validation target template
├── saas-quickstart/
│   └── SKILL.md                             # 5-step launch guide
├── nextjs-fullstack/
│   └── SKILL.md                             # App Router patterns
├── supabase-developer/
│   └── SKILL.md                             # Supabase full reference
├── stripe-payments/
│   └── SKILL.md                             # Stripe beginner guide
├── mvp-billing-system/
│   └── SKILL.md                             # Production billing (EN)
├── mvp-billing-system-cn/
│   └── SKILL.md                             # Production billing (CN)
├── supabase-gemini-deploy/
│   └── SKILL.md                             # Edge Function troubleshooting
└── deploy-gate/
    └── SKILL.md                             # 6-gate deployment checks
```

## ❓ FAQ

**Q: Do I need all 9 skills?**
> No. Each skill works independently. Start with what you need now. The `mvp-billing-system` + `deploy-gate` combo covers the most ground.

**Q: Will these conflict with other Claude Code skills?**
> No. Skills are loaded on-demand based on keyword matching. They don't interfere with each other or with skills from other sources.

**Q: Can I use these with a different tech stack?**
> The architectural patterns (idempotency, webhook handling, credit systems) are stack-agnostic. Code templates target Next.js + Supabase + Stripe but can be adapted.

**Q: How do I update?**
> ```bash
> cd saas-skills-suite && git pull
> cp -r */* ~/.claude/skills/
> ```

**Q: I found an issue / want to contribute.**
> Open an [Issue](https://github.com/Alex647648/saas-skills-suite/issues) or [Pull Request](https://github.com/Alex647648/saas-skills-suite/pulls). Contributions welcome!

## 📄 License

[MIT](./LICENSE) — Use freely, modify freely, no attribution required.
