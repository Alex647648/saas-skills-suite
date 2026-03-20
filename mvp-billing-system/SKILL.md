---
name: mvp-billing-system
description: |
  Complete SaaS billing system implementation guide — from database schema to production.
  Covers Supabase + Stripe + Next.js with dual payment (subscription + one-time),
  dual wallet (subscription credits + top-up credits), credit-based billing,
  webhook debugging, and 8 real-world pitfalls with solutions.
  Activate when: "billing system", "payment integration", "Stripe setup",
  "subscription management", "credit system", "webhook debugging",
  "SaaS payments", "checkout flow", "pricing page"
---

# MVP Billing System — From Schema to Production

> A battle-tested implementation guide distilled from 4 days, 32 commits building
> a production SaaS billing system (Next.js 15 + Supabase + Stripe).

## When to Activate

- Building a new SaaS product that needs subscription + credit billing
- Integrating Stripe with Supabase Edge Functions
- Implementing dual payment rails (credit card + WeChat/Alipay)
- Designing a credit/wallet system with idempotency guarantees
- Debugging webhook failures or payment inconsistencies

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        FRONTEND (Next.js)                       │
│  PricingPanel → PaymentMethodModal → billingService.ts          │
│  SubscriptionCard / PaywallModal / CreditBadge / BillingHistory │
└──────────┬──────────────────────────────────┬───────────────────┘
           │ Bearer token                     │ Bearer token
           ▼                                  ▼
┌─────────────────────┐          ┌────────────────────────────────┐
│  pay-create-checkout │          │  pay-manage-subscription       │
│  (3 flows:           │          │  (cancel/resume/change/portal) │
│   subscription,      │          └────────────────────────────────┘
│   one_time, topup)   │
└──────────┬───────────┘
           │ Stripe Checkout URL
           ▼
┌─────────────────────┐    webhook     ┌──────────────────────────┐
│   Stripe Checkout    │──────────────→│  pay-stripe-webhook       │
│   (hosted page)      │               │  (signature verify,       │
└──────────────────────┘               │   event dedup,            │
                                       │   multi-event dispatch)   │
                                       └──────────┬───────────────┘
                                                  │ RPC calls
                                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                     SUPABASE (PostgreSQL)                        │
│  member_profile ←→ billing_wallet ←→ billing_wallet_tx          │
│  pay_subscription    pay_checkout_session                        │
│  stripe_webhook_events   rate_limit_log   admin_audit_log       │
│                                                                  │
│  RPCs: billing_deduct_credits, billing_refund_credits,          │
│        billing_grant_subscription_credits,                       │
│        billing_activate_manual_subscription,                     │
│        billing_renew_manual_subscription,                        │
│        billing_topup_credits, check_rate_limit                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Database Schema (Day 1)

### Core Tables

Design principle: **1:1 relationships** for profile and wallet, append-only for transactions.

#### 1. member_profile — User billing profile

```sql
CREATE TABLE member_profile (
  user_id     UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  tier        TEXT NOT NULL DEFAULT 'free'
              CHECK (tier IN ('free','basic','pro','max')),
  free_quota  INT  NOT NULL DEFAULT 3
              CHECK (free_quota >= 0 AND free_quota <= 100),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

#### 2. billing_wallet — Credit wallet (dual bucket)

```sql
CREATE TABLE billing_wallet (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL UNIQUE REFERENCES auth.users(id) ON DELETE CASCADE,
  balance         NUMERIC(10,1) NOT NULL DEFAULT 0     -- subscription credits (reset monthly)
                  CHECK (balance >= 0),
  topup_balance   NUMERIC(10,1) NOT NULL DEFAULT 0     -- purchased credits (permanent)
                  CHECK (topup_balance >= 0),
  freeze_amount   NUMERIC(10,1) NOT NULL DEFAULT 0
                  CHECK (freeze_amount >= 0),
  total_consumed  NUMERIC(10,1) NOT NULL DEFAULT 0,
  total_recharged NUMERIC(10,1) NOT NULL DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Key design: **dual bucket wallet**
- `balance` = subscription credits, reset each billing period
- `topup_balance` = purchased credits, never expire
- Deduction priority: `balance` first (expires sooner), then `topup_balance`

#### 3. billing_wallet_tx — Transaction ledger (append-only)

```sql
CREATE TABLE billing_wallet_tx (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tx_no         TEXT NOT NULL UNIQUE
                CHECK (tx_no ~ '^WT-[0-9]{14}-[A-Z0-9]{4,8}$'),
  wallet_id     UUID NOT NULL REFERENCES billing_wallet(id),
  user_id       UUID NOT NULL REFERENCES auth.users(id),
  biz_type      TEXT NOT NULL
                CHECK (biz_type IN (
                  'SUBSCRIPTION_GRANT','USAGE','USAGE_REFUND',
                  'FREE_TRIAL','ADMIN_ADJUST','RECHARGE',
                  'RECHARGE_REFUND','FREEZE','UNFREEZE'
                )),
  biz_id        TEXT,                    -- external reference for idempotency
  title         TEXT NOT NULL,
  amount        NUMERIC(10,1) NOT NULL,  -- positive = credit, negative = debit
  balance_after NUMERIC(10,1) NOT NULL CHECK (balance_after >= 0),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wallet_tx_user    ON billing_wallet_tx(user_id, created_at DESC);
CREATE INDEX idx_wallet_tx_biztype ON billing_wallet_tx(biz_type, created_at DESC);
```

#### 4. pay_subscription — Subscription records

```sql
CREATE TABLE pay_subscription (
  id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id                 UUID NOT NULL REFERENCES auth.users(id),
  stripe_subscription_id  TEXT NOT NULL UNIQUE,
  stripe_price_id         TEXT NOT NULL,
  stripe_customer_id      TEXT NOT NULL,
  tier                    TEXT NOT NULL,
  billing_interval        TEXT NOT NULL CHECK (billing_interval IN ('month','year')),
  status                  TEXT NOT NULL CHECK (status IN ('ACTIVE','PAST_DUE','CANCELED','EXPIRED')),
  current_period_start    TIMESTAMPTZ NOT NULL,
  current_period_end      TIMESTAMPTZ NOT NULL,
  cancel_at_period_end    BOOLEAN NOT NULL DEFAULT false,
  auto_renew              BOOLEAN NOT NULL DEFAULT true,  -- false = manual (WeChat/Alipay)
  last_grant_period_start TIMESTAMPTZ,    -- idempotency: last credit grant period
  last_grant_tier         TEXT,           -- idempotency: last credit grant tier
  created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Prevent duplicate active manual subscriptions
CREATE UNIQUE INDEX idx_unique_active_manual_sub
  ON pay_subscription (user_id)
  WHERE auto_renew = false AND status IN ('ACTIVE', 'PAST_DUE');
```

#### 5. stripe_webhook_events — Webhook deduplication

```sql
CREATE TABLE stripe_webhook_events (
  id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  stripe_event_id TEXT NOT NULL UNIQUE,
  event_type      TEXT NOT NULL,
  processed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### New User Trigger

Auto-create profile + wallet with initial credits on signup:

```sql
CREATE OR REPLACE FUNCTION billing_handle_new_user()
RETURNS TRIGGER
LANGUAGE plpgsql SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  INSERT INTO member_profile (user_id) VALUES (NEW.id)
    ON CONFLICT (user_id) DO NOTHING;
  INSERT INTO billing_wallet (user_id, balance) VALUES (NEW.id, 30)
    ON CONFLICT (user_id) DO NOTHING;
  RETURN NEW;
END;
$$;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION billing_handle_new_user();
```

### RLS Policies

```sql
-- Users can only read their own data
ALTER TABLE member_profile ENABLE ROW LEVEL SECURITY;
CREATE POLICY "users read own profile"
  ON member_profile FOR SELECT USING (auth.uid() = user_id);

ALTER TABLE billing_wallet ENABLE ROW LEVEL SECURITY;
CREATE POLICY "users read own wallet"
  ON billing_wallet FOR SELECT USING (auth.uid() = user_id);

ALTER TABLE billing_wallet_tx ENABLE ROW LEVEL SECURITY;
CREATE POLICY "users read own tx"
  ON billing_wallet_tx FOR SELECT USING (auth.uid() = user_id);

ALTER TABLE pay_subscription ENABLE ROW LEVEL SECURITY;
CREATE POLICY "users read own sub"
  ON pay_subscription FOR SELECT USING (auth.uid() = user_id);
```

---

## Phase 2: Billing RPC Functions (Day 1)

All RPCs use `SECURITY DEFINER` + `SET search_path = public` for privilege escalation safety.

### Transaction Number Generator

```sql
CREATE OR REPLACE FUNCTION billing_generate_tx_no()
RETURNS TEXT LANGUAGE plpgsql AS $$
BEGIN
  RETURN 'WT-' || to_char(now(), 'YYYYMMDDHH24MISS')
       || '-' || upper(substr(replace(gen_random_uuid()::text, '-', ''), 1, 8));
END;
$$;
```

### billing_deduct_credits — Atomic dual-bucket deduction

```sql
CREATE OR REPLACE FUNCTION billing_deduct_credits(
  p_user_id UUID, p_amount NUMERIC(10,1),
  p_action TEXT, p_model TEXT, p_biz_id TEXT DEFAULT NULL
)
RETURNS TABLE(ok BOOLEAN, new_balance NUMERIC(10,1), free_remaining INT, error_code TEXT)
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, extensions
AS $$
DECLARE
  v_profile member_profile%ROWTYPE;
  v_wallet  billing_wallet%ROWTYPE;
  v_total   NUMERIC(10,1);
  v_from_balance NUMERIC(10,1);
  v_from_topup   NUMERIC(10,1);
  v_new_balance  NUMERIC(10,1);
  v_new_topup    NUMERIC(10,1);
  v_tx_no   TEXT;
  v_updated INT;
BEGIN
  -- Input validation
  IF p_amount <= 0 THEN
    RETURN QUERY SELECT false, 0::NUMERIC(10,1), 0, 'INVALID_AMOUNT'::TEXT; RETURN;
  END IF;

  -- Lock profile + wallet (prevents concurrent deductions)
  SELECT * INTO v_profile FROM member_profile WHERE user_id = p_user_id FOR UPDATE;
  IF NOT FOUND THEN
    RETURN QUERY SELECT false, 0::NUMERIC(10,1), 0, 'USER_NOT_FOUND'::TEXT; RETURN;
  END IF;

  SELECT * INTO v_wallet FROM billing_wallet WHERE user_id = p_user_id FOR UPDATE;

  -- Check total available
  v_total := v_wallet.balance + v_wallet.topup_balance;
  IF v_total < p_amount THEN
    RETURN QUERY SELECT false, v_total, 0, 'INSUFFICIENT_CREDITS'::TEXT; RETURN;
  END IF;

  -- Deduction priority: balance first (expires sooner), then topup
  IF v_wallet.balance >= p_amount THEN
    v_from_balance := p_amount;  v_from_topup := 0;
  ELSE
    v_from_balance := v_wallet.balance;
    v_from_topup   := p_amount - v_wallet.balance;
  END IF;

  v_new_balance := v_wallet.balance - v_from_balance;
  v_new_topup   := v_wallet.topup_balance - v_from_topup;

  -- Optimistic lock update
  UPDATE billing_wallet
    SET balance = v_new_balance, topup_balance = v_new_topup,
        total_consumed = total_consumed + p_amount, updated_at = now()
    WHERE id = v_wallet.id
      AND balance >= v_from_balance AND topup_balance >= v_from_topup;

  GET DIAGNOSTICS v_updated = ROW_COUNT;
  IF v_updated = 0 THEN
    RETURN QUERY SELECT false, v_total, 0, 'INSUFFICIENT_CREDITS'::TEXT; RETURN;
  END IF;

  -- Write transaction log
  v_tx_no := billing_generate_tx_no();
  INSERT INTO billing_wallet_tx (tx_no, wallet_id, user_id, biz_type, biz_id, title, amount, balance_after)
    VALUES (v_tx_no, v_wallet.id, p_user_id, 'USAGE', p_biz_id,
            LEFT(p_action, 50) || ' · ' || LEFT(p_model, 50),
            -p_amount, v_new_balance + v_new_topup);

  RETURN QUERY SELECT true, (v_new_balance + v_new_topup), 0, NULL::TEXT;
END;
$$;
```

### billing_refund_credits — Idempotent refund

```sql
CREATE OR REPLACE FUNCTION billing_refund_credits(
  p_user_id UUID, p_amount NUMERIC(10,1),
  p_biz_id TEXT, p_reason TEXT DEFAULT 'generation failed'
)
RETURNS VOID LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, extensions
AS $$
DECLARE v_wallet billing_wallet%ROWTYPE; v_tx_no TEXT; v_new_bal NUMERIC(10,1);
BEGIN
  IF p_amount <= 0 THEN RAISE EXCEPTION 'INVALID_AMOUNT'; END IF;

  -- Idempotency: skip if already refunded for this biz_id
  IF EXISTS (SELECT 1 FROM billing_wallet_tx
    WHERE biz_id = p_biz_id AND biz_type = 'USAGE_REFUND' AND user_id = p_user_id
  ) THEN RETURN; END IF;

  -- Verify matching USAGE tx exists (refuse phantom refunds)
  IF NOT EXISTS (SELECT 1 FROM billing_wallet_tx
    WHERE biz_id = p_biz_id AND biz_type = 'USAGE' AND user_id = p_user_id
  ) THEN RETURN; END IF;

  SELECT * INTO v_wallet FROM billing_wallet WHERE user_id = p_user_id FOR UPDATE;
  IF NOT FOUND THEN RETURN; END IF;

  v_new_bal := v_wallet.balance + p_amount;
  UPDATE billing_wallet
    SET balance = v_new_bal, total_consumed = GREATEST(total_consumed - p_amount, 0),
        updated_at = now()
    WHERE id = v_wallet.id;

  v_tx_no := billing_generate_tx_no();
  INSERT INTO billing_wallet_tx (tx_no, wallet_id, user_id, biz_type, biz_id, title, amount, balance_after)
    VALUES (v_tx_no, v_wallet.id, p_user_id, 'USAGE_REFUND', p_biz_id,
            'refund: ' || LEFT(p_reason, 200), p_amount, v_new_bal + v_wallet.topup_balance);
END;
$$;
```

### billing_grant_subscription_credits — Composite idempotency

```sql
CREATE OR REPLACE FUNCTION billing_grant_subscription_credits(
  p_user_id UUID, p_tier TEXT, p_period_start TIMESTAMPTZ
)
RETURNS BOOLEAN LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, extensions
AS $$
DECLARE
  v_wallet billing_wallet%ROWTYPE; v_sub pay_subscription%ROWTYPE;
  v_grant NUMERIC(10,1); v_tx_no TEXT; v_new_bal NUMERIC(10,1); v_real_tier TEXT;
BEGIN
  -- Lock subscription
  SELECT * INTO v_sub FROM pay_subscription
    WHERE user_id = p_user_id AND status = 'ACTIVE' FOR UPDATE;
  IF NOT FOUND THEN RETURN false; END IF;

  -- Derive tier from LOCKED row (never trust caller-supplied tier)
  v_real_tier := v_sub.tier;

  -- Composite idempotency: same period + same tier = already granted
  IF v_sub.last_grant_period_start IS NOT NULL
     AND v_sub.last_grant_period_start = p_period_start
     AND v_sub.last_grant_tier = v_real_tier THEN
    RETURN false;
  END IF;

  -- Determine grant amount
  v_grant := CASE v_real_tier
    WHEN 'basic' THEN 1000 WHEN 'pro' THEN 5000 WHEN 'max' THEN 20000 ELSE 0
  END;
  IF v_grant = 0 THEN RETURN false; END IF;

  -- Update wallet
  -- Same period (upgrade) → accumulate | New period (renewal) → reset
  SELECT * INTO v_wallet FROM billing_wallet WHERE user_id = p_user_id FOR UPDATE;
  IF v_sub.last_grant_period_start = p_period_start THEN
    v_new_bal := v_wallet.balance + v_grant;   -- accumulate (upgrade)
  ELSE
    v_new_bal := v_grant;                      -- reset (new period)
  END IF;

  UPDATE billing_wallet
    SET balance = v_new_bal, total_recharged = total_recharged + v_grant, updated_at = now()
    WHERE id = v_wallet.id;

  -- Mark grant period AND tier (for composite idempotency)
  UPDATE pay_subscription
    SET last_grant_period_start = p_period_start, last_grant_tier = v_real_tier, updated_at = now()
    WHERE id = v_sub.id;

  UPDATE member_profile SET tier = v_real_tier, updated_at = now() WHERE user_id = p_user_id;

  v_tx_no := billing_generate_tx_no();
  INSERT INTO billing_wallet_tx (tx_no, wallet_id, user_id, biz_type, biz_id, title, amount, balance_after)
    VALUES (v_tx_no, v_wallet.id, p_user_id, 'SUBSCRIPTION_GRANT', v_sub.id::TEXT,
            v_real_tier || ' monthly credits', v_grant, v_new_bal + v_wallet.topup_balance);
  RETURN true;
END;
$$;
```

### billing_topup_credits — Top-up purchase

```sql
CREATE OR REPLACE FUNCTION billing_topup_credits(
  p_user_id UUID, p_amount NUMERIC(10,1), p_checkout_session_id TEXT
)
RETURNS BOOLEAN LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, extensions
AS $$
DECLARE v_wallet billing_wallet%ROWTYPE; v_tx_no TEXT; v_new_topup NUMERIC(10,1);
BEGIN
  IF p_amount <= 0 THEN RAISE EXCEPTION 'INVALID_AMOUNT'; END IF;

  -- Idempotency: skip if already processed
  IF EXISTS (SELECT 1 FROM billing_wallet_tx
    WHERE user_id = p_user_id AND biz_type = 'RECHARGE' AND biz_id = p_checkout_session_id
  ) THEN RETURN true; END IF;

  SELECT * INTO v_wallet FROM billing_wallet WHERE user_id = p_user_id FOR UPDATE;
  v_new_topup := v_wallet.topup_balance + p_amount;

  UPDATE billing_wallet
    SET topup_balance = v_new_topup, total_recharged = total_recharged + p_amount, updated_at = now()
    WHERE id = v_wallet.id;

  v_tx_no := billing_generate_tx_no();
  INSERT INTO billing_wallet_tx (tx_no, wallet_id, user_id, biz_type, biz_id, title, amount, balance_after)
    VALUES (v_tx_no, v_wallet.id, p_user_id, 'RECHARGE', p_checkout_session_id,
            'Credit top-up +' || p_amount::TEXT, p_amount, v_wallet.balance + v_new_topup);
  RETURN true;
END;
$$;
```

---

## Phase 3: Edge Functions — Stripe Integration (Day 2)

### Shared Module Pattern (`_shared/`)

#### `_shared/cors.ts` — CORS + JSON response helpers

```typescript
export function corsHeaders(req?: Request): Record<string, string> {
  const origin = req?.headers.get('origin') ?? '';
  const allowed = Deno.env.get('ALLOWED_ORIGIN') ?? 'https://your-app.com';
  const allowedOrigins = allowed === '*'
    ? ['*']
    : [allowed, allowed.replace('https://', 'https://www.')];
  const isAllowed = allowed === '*' || allowedOrigins.includes(origin);
  return {
    'Access-Control-Allow-Origin': isAllowed ? (allowed === '*' ? '*' : origin) : 'null',
    'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
    'Access-Control-Allow-Methods': 'POST, OPTIONS',
    Vary: 'Origin',
    'Content-Type': 'application/json',
  };
}

export function jsonResponse(body: Record<string, unknown>, status = 200, req?: Request): Response {
  return new Response(JSON.stringify(body), { status, headers: corsHeaders(req) });
}
```

#### `_shared/pricing.ts` — Single source of truth for prices

```typescript
// Map Stripe price IDs → internal tier/interval
export const PRICE_TO_TIER: Record<string, string> = {
  'price_basic_monthly':  'basic',
  'price_basic_yearly':   'basic',
  'price_pro_monthly':    'pro',
  'price_pro_yearly':     'pro',
  'price_max_monthly':    'max',
  'price_max_yearly':     'max',
};

export const PRICE_TO_INTERVAL: Record<string, string> = {
  'price_basic_monthly': 'month',  'price_basic_yearly': 'year',
  'price_pro_monthly':   'month',  'price_pro_yearly':   'year',
  'price_max_monthly':   'month',  'price_max_yearly':   'year',
};

export const ALLOWED_PRICE_IDS: string[] = Object.keys(PRICE_TO_TIER);

// Expected amounts (smallest currency unit) for webhook cross-validation
export const EXPECTED_AMOUNTS: Record<string, Record<string, number>> = {
  basic: { month: 2900, year: 24900 },
  pro:   { month: 7900, year: 69900 },
  max:   { month: 19900, year: 179900 },
};

// Top-up credit packs
export const TOPUP_PACKS: Record<string, { credits: number; amount: number; name: string }> = {
  small:  { credits: 800,  amount: 2900,  name: 'Small Pack · 800 credits' },
  medium: { credits: 4000, amount: 7900,  name: 'Medium Pack · 4000 credits' },
  large:  { credits: 8000, amount: 10000, name: 'Large Pack · 8000 credits' },
};

// Stripe status → internal status mapping
export const STRIPE_STATUS_MAP: Record<string, string> = {
  active: 'ACTIVE', trialing: 'ACTIVE',
  past_due: 'PAST_DUE', unpaid: 'PAST_DUE', incomplete: 'PAST_DUE',
  incomplete_expired: 'EXPIRED', canceled: 'CANCELED', paused: 'PAST_DUE',
};
```

### pay-create-checkout — Three checkout flows

```typescript
// Edge Function: pay-create-checkout/index.ts
// Handles 3 payment flows: subscription, one_time, topup

Deno.serve(async (req: Request) => {
  // 1. CORS preflight
  if (req.method === 'OPTIONS') return new Response('ok', { headers: corsHeaders(req) });

  // 2. Payload size limit
  const contentLength = parseInt(req.headers.get('content-length') ?? '0', 10);
  if (contentLength > 65_536) return jsonResponse({ ok: false, error_code: 'PAYLOAD_TOO_LARGE' }, 413, req);

  // 3. Auth: use getUser(token), NOT JWT gateway verification
  const token = req.headers.get('Authorization')?.replace('Bearer ', '');
  const { data: { user } } = await supabase.auth.getUser(token);

  // 4. Database-backed rate limiting
  const { data: rateLimitOk } = await supabase.rpc('check_rate_limit', {
    p_user_id: user.id, p_action: 'checkout', p_max: 10, p_window_seconds: 60,
  });

  // 5. Parse request and validate
  const { price_id, return_path, payment_flow, is_renewal, topup_pack_id } = await req.json();

  // 6. Block duplicate subscriptions (same tier + interval)
  // 7. Block downgrades for manual subscribers

  // 8. Get or create Stripe customer
  // 9. Build checkout URL server-side (NEVER from client origin)
  const appBaseUrl = Deno.env.get('APP_BASE_URL');

  // 10. Branch by payment_flow:
  switch (payment_flow) {
    case 'topup':
      // mode: 'payment', payment_method_types: ['wechat_pay', 'alipay']
      // metadata: { payment_flow: 'topup', topup_pack_id, topup_credits }
      break;
    case 'one_time':
      // mode: 'payment', payment_method_types: ['wechat_pay', 'alipay']
      // metadata: { payment_flow: 'one_time', tier, billing_interval, is_renewal }
      break;
    default: // 'subscription'
      // mode: 'subscription', line_items: [{ price: price_id }]
      // metadata: { supabase_user_id } on BOTH session and subscription_data
  }

  // 11. Record in pay_checkout_session table
  return jsonResponse({ ok: true, checkout_url: session.url }, 200, req);
});
```

### pay-stripe-webhook — Robust event handler

```typescript
// Critical: manual HMAC-SHA256 verification for Deno
async function verifyStripeSignature(payload: string, sigHeader: string, secret: string): Promise<boolean> {
  const parts = sigHeader.split(',').reduce<Record<string, string>>((acc, part) => {
    const [k, v] = part.split('=');
    acc[k] = v;
    return acc;
  }, {});
  const timestamp = parts['t'];
  const signature = parts['v1'];
  if (!timestamp || !signature) return false;

  // Reject events older than 5 minutes
  if (Math.abs(Math.floor(Date.now() / 1000) - parseInt(timestamp)) > 300) return false;

  const signedPayload = `${timestamp}.${payload}`;
  const key = await crypto.subtle.importKey(
    'raw', new TextEncoder().encode(secret),
    { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']
  );
  const sig = await crypto.subtle.sign('HMAC', key, new TextEncoder().encode(signedPayload));
  const computed = Array.from(new Uint8Array(sig)).map(b => b.toString(16).padStart(2, '0')).join('');
  return computed === signature;
}

// Handle Stripe API version changes for period fields
function extractPeriod(rawSub: Record<string, any>): { periodStart: string; periodEnd: string } {
  const cp = rawSub.current_period;
  const startRaw = cp?.start ?? rawSub.current_period_start;  // new vs old API
  const endRaw = cp?.end ?? rawSub.current_period_end;
  // Convert unix timestamps or ISO strings
  return { periodStart: toISO(startRaw), periodEnd: toISO(endRaw) };
}

// Event handling flow:
// 1. Verify signature (raw body, NOT parsed JSON)
// 2. Event deduplication via stripe_webhook_events table
// 3. Claim event (INSERT before processing)
// 4. Switch on event.type:
//    - checkout.session.completed → handle topup / one_time / subscription
//    - customer.subscription.created/updated → upsert sub + grant credits
//    - invoice.paid → grant credits for renewal
//    - customer.subscription.deleted → expire sub + downgrade tier
// 5. Return 500 on processing errors (Stripe retries up to 3 days)
```

### pay-manage-subscription — 4 actions

```typescript
// Actions: cancel, resume, change_plan, portal
// Key pattern: check if subscription is manual (auto_renew === false)
//   Manual subs: update DB directly (no Stripe API needed)
//   Auto-renew subs: call Stripe API then update DB

switch (action) {
  case 'cancel':
    // Manual: just update status to 'CANCELED' in DB
    // Auto: stripe.subscriptions.update({ cancel_at_period_end: true })
    break;
  case 'resume':
    // Manual: can only resume if period hasn't expired
    // Auto: stripe.subscriptions.update({ cancel_at_period_end: false })
    break;
  case 'change_plan':
    // Manual: NOT SUPPORTED (must wait for expiry)
    // Auto: stripe.subscriptions.update({ items: [{ id, price: new_price_id }] })
    break;
  case 'portal':
    // Manual: NOT SUPPORTED
    // Auto: stripe.billingPortal.sessions.create()
    break;
}
```

---

## Phase 4: Frontend Billing Components (Day 2)

### Type Definitions

```typescript
export type BillingTier = 'free' | 'basic' | 'pro' | 'max';
export type BillingInterval = 'month' | 'year';
export type SubscriptionStatus = 'ACTIVE' | 'PAST_DUE' | 'CANCELED' | 'EXPIRED';
export type PaymentFlow = 'subscription' | 'one_time' | 'topup';

export interface PlanOption {
  readonly tier: Exclude<BillingTier, 'free'>;
  readonly name: string;
  readonly monthlyPrice: number;     // in smallest currency unit
  readonly yearlyPrice: number;
  readonly monthlyCredits: number;
  readonly models: string[];
  readonly stripePriceIds: { readonly monthly: string; readonly yearly: string };
}
```

### billingService — Frontend API layer

```typescript
const FUNCTION_URL = process.env.NEXT_PUBLIC_SUPABASE_URL + '/functions/v1';

async function invokeFunction<T>(fnName: string, body: Record<string, unknown>): Promise<T> {
  const { data: { session } } = await supabase.auth.getSession();
  if (!session) throw new Error('Not authenticated');

  const res = await fetch(`${FUNCTION_URL}/${fnName}`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${session.access_token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(body),
  });
  const result = await res.json();
  if (!res.ok || result.ok === false) throw new Error(result.message || `Request failed: ${res.status}`);
  return result as T;
}

export const billingService = {
  getProfile:       () => supabase.from('member_profile').select('*').single(),
  getWallet:        () => supabase.from('billing_wallet').select('*').single(),
  getTransactions:  (limit, offset) => supabase.from('billing_wallet_tx').select('*').order('created_at', { ascending: false }).range(offset, offset + limit - 1),
  getActiveSubscription: () => supabase.from('pay_subscription').select('*').in('status', ['ACTIVE','PAST_DUE','CANCELED']).order('created_at', { ascending: false }).limit(1).maybeSingle(),
  createCheckout:   (priceId, flow, isRenewal) => invokeFunction('pay-create-checkout', { price_id: priceId, return_path: window.location.pathname, payment_flow: flow, is_renewal: isRenewal }),
  cancelSubscription: () => invokeFunction('pay-manage-subscription', { action: 'cancel' }),
  resumeSubscription: () => invokeFunction('pay-manage-subscription', { action: 'resume' }),
  changePlan:       (newPriceId) => invokeFunction('pay-manage-subscription', { action: 'change_plan', new_price_id: newPriceId }),
  topUpCredits:     (packId) => invokeFunction('pay-create-checkout', { return_path: window.location.pathname, payment_flow: 'topup', topup_pack_id: packId }),
  getPortalUrl:     () => invokeFunction('pay-manage-subscription', { action: 'portal' }),
} as const;
```

### Component Hierarchy

```
AccountDashboard
├── CreditBadge          — real-time credit display (balance + topup_balance)
├── SubscriptionCard     — status + management (cancel/resume/renew)
├── PricingPanel         — tier selection modal
│   └── PaymentMethodModal — bank card vs WeChat/Alipay
├── PaywallModal         — triggered when credits insufficient
│   └── TopUpOptions     — quick credit top-up packs
├── BillingHistory       — transaction log + payment records
└── ArticleArchiveList   — generated article history
```

### PricingPanel — Key patterns

```typescript
// 1. Upgrade/downgrade logic
const handleSubscribe = (plan: PlanOption) => {
  const currentRank = TIER_RANK[currentTier];
  const planRank = TIER_RANK[plan.tier];

  // Block downgrade for manual subscribers
  if (isManualSub && planRank < currentRank) return;
  // Block same-tier-same-interval
  if (isManualSub && planRank === currentRank && interval === currentInterval) return;

  if (currentTier !== 'free' && !isManualSub) {
    // Auto-renew: use Stripe change_plan API
    handleChangePlan(plan);
  } else {
    // Free or manual: show payment method modal
    setPendingPlan(plan);
  }
};

// 2. Payment method selection → checkout flow
const handlePaymentMethodSelect = async (flow: PaymentFlow) => {
  const url = await billingService.createCheckout(priceId, flow);
  window.location.href = url;  // redirect to Stripe Checkout
};
```

---

## Phase 5: Advanced — Dual Payment & Dual Wallet (Day 3)

### Manual Subscription Design (WeChat/Alipay)

Manual subscriptions (`auto_renew: false`) for payment methods that don't support recurring billing:

```
Flow: User → PaymentMethodModal → "WeChat Pay" → one_time checkout
     → Stripe payment → webhook checkout.session.completed
     → billing_activate_manual_subscription (DB creates sub + grants credits)
```

Lifecycle: **activate → cancel → resume → expire → renew**

- Cancel: just sets status to 'CANCELED' (still usable until period end)
- Resume: sets status back to 'ACTIVE' (only if within period)
- Expire: automatic via pg_cron job or checked at read time
- Renew: new one-time payment → billing_renew_manual_subscription

### Wallet Split: balance vs topup_balance

```
┌─────────────────────────────────┐
│         billing_wallet          │
│  ┌──────────┐  ┌──────────────┐│
│  │ balance  │  │ topup_balance││
│  │(sub grant│  │(purchased,   ││
│  │ resets   │  │ permanent)   ││
│  │ monthly) │  │              ││
│  └──────────┘  └──────────────┘│
│       ▲                ▲       │
│       │                │       │
│  subscription      top-up      │
│  grant RPC       purchase RPC  │
└─────────────────────────────────┘

Deduction priority: balance → topup_balance
Refund target: always balance (subscription credits)
Subscription grant: resets balance only, topup_balance untouched
```

### Downgrade Prevention (3-layer defense)

```
Layer 1: Frontend — PricingPanel disables downgrade buttons
Layer 2: Edge Function — pay-create-checkout checks TIER_RANK before creating checkout
Layer 3: Database — billing_activate_manual_subscription raises DOWNGRADE_BLOCKED exception
```

```sql
-- Layer 3: Database guard (final defense)
SELECT * INTO v_old_sub FROM pay_subscription
  WHERE user_id = p_user_id AND auto_renew = false
    AND status IN ('ACTIVE','PAST_DUE','CANCELED')
    AND current_period_end > now()
  FOR UPDATE;

IF FOUND THEN
  v_old_rank := CASE v_old_sub.tier WHEN 'basic' THEN 1 WHEN 'pro' THEN 2 WHEN 'max' THEN 3 ELSE 0 END;
  v_new_rank := CASE p_tier WHEN 'basic' THEN 1 WHEN 'pro' THEN 2 WHEN 'max' THEN 3 ELSE 0 END;
  IF v_new_rank < v_old_rank THEN
    RAISE EXCEPTION 'DOWNGRADE_BLOCKED: cannot downgrade from % to % mid-cycle', v_old_sub.tier, p_tier;
  END IF;
END IF;
```

---

## Phase 6: Security & Production Hardening (Day 4)

### Webhook Event Deduplication

```sql
-- Table: stripe_webhook_events (UNIQUE on stripe_event_id)
-- Pattern: INSERT before processing, skip if unique constraint fails

const { data: existingEvent } = await supabase
  .from('stripe_webhook_events')
  .select('id').eq('stripe_event_id', event.id).maybeSingle();
if (existingEvent) return; // already processed

const { error: insertErr } = await supabase.from('stripe_webhook_events').insert({
  stripe_event_id: event.id, event_type: event.type,
});
if (insertErr) return; // another instance claimed it
```

### Amount Cross-Validation

```typescript
// In webhook: verify metadata tier/interval matches the checkout record
const { data: csRecord } = await supabase
  .from('pay_checkout_session')
  .select('stripe_price_id')
  .eq('stripe_session_id', session.id)
  .maybeSingle();

if (csRecord?.stripe_price_id) {
  const expectedTier = PRICE_TO_TIER[csRecord.stripe_price_id];
  if (tier !== expectedTier) {
    console.error('metadata mismatch!');
    break; // abort processing
  }
}

// Also verify actual payment amount
const expectedAmount = EXPECTED_AMOUNTS[tier]?.[billingInterval];
if (expectedAmount && actualAmount < expectedAmount) {
  console.error('payment amount mismatch!');
  break;
}
```

### Database-Backed Rate Limiting

```sql
CREATE TABLE rate_limit_log (
  id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id    UUID NOT NULL,
  action     TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rate_limit_log_lookup ON rate_limit_log(user_id, action, created_at DESC);
ALTER TABLE rate_limit_log ENABLE ROW LEVEL SECURITY;

CREATE OR REPLACE FUNCTION check_rate_limit(
  p_user_id UUID, p_action TEXT, p_max INT, p_window_seconds INT
) RETURNS BOOLEAN LANGUAGE plpgsql SECURITY DEFINER SET search_path = public AS $$
DECLARE v_count INT;
BEGIN
  SELECT count(*) INTO v_count FROM rate_limit_log
    WHERE user_id = p_user_id AND action = p_action
      AND created_at > now() - (p_window_seconds || ' seconds')::interval;
  IF v_count >= p_max THEN RETURN false; END IF;
  INSERT INTO rate_limit_log(user_id, action) VALUES (p_user_id, p_action);
  RETURN true;
END;
$$;
```

Advantage over in-memory rate limiting: **survives cold starts and works across multiple Deno isolates**.

### Admin Audit Logging

```sql
CREATE TABLE admin_audit_log (
  id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  admin_id   UUID NOT NULL REFERENCES auth.users(id),
  action     TEXT NOT NULL,
  target_uid UUID NOT NULL,
  payload    JSONB NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
ALTER TABLE admin_audit_log ENABLE ROW LEVEL SECURITY;
-- No RLS policies = only service_role can read/write
```

### pg_cron Cleanup Jobs

```sql
-- Clean webhook events > 30 days
SELECT cron.schedule('cleanup_webhook_events', '0 3 * * *',
  $$DELETE FROM stripe_webhook_events WHERE processed_at < now() - interval '30 days'$$);

-- Clean rate limit logs > 7 days
SELECT cron.schedule('cleanup_rate_limit_log', '5 3 * * *',
  $$DELETE FROM rate_limit_log WHERE created_at < now() - interval '7 days'$$);
```

---

## Pitfall Encyclopedia

### PITFALL-1: Stripe API Version Hell

**Problem**: Stripe regularly introduces breaking changes. The 2025-02 update removed `current_period_start`/`current_period_end` from the subscription object, replacing them with a `current_period` object `{ start, end }`.

**Solution**: Build a version-agnostic extractor:
```typescript
function extractPeriod(rawSub: Record<string, any>) {
  const cp = rawSub.current_period;
  const start = cp?.start ?? rawSub.current_period_start;
  const end = cp?.end ?? rawSub.current_period_end;
  return { periodStart: toISO(start), periodEnd: toISO(end) };
}
```

### PITFALL-2: Deno Edge Function JWT (ES256 vs HS256)

**Problem**: Supabase Auth signs user JWTs with **ES256** (asymmetric), but the Edge Function API Gateway only verifies **HS256** (symmetric, using anon key). Result: all user requests get 401 at the gateway level.

**Symptom**: Fast 401 response (< 100ms) = gateway rejection.

**Solution**: Deploy with `--no-verify-jwt` and verify auth in function code:
```bash
supabase functions deploy my-function --no-verify-jwt
```
```typescript
// In function code:
const { data: { user } } = await supabase.auth.getUser(token);
```

### PITFALL-3: Duplicate Subscription on Plan Change

**Problem**: Stripe `customer.subscription.updated` fires when changing plans. If you use `INSERT` instead of `UPSERT`, you create duplicate subscription records.

**Solution**: Always UPSERT on `stripe_subscription_id`:
```typescript
await supabase.from('pay_subscription')
  .upsert(payload, { onConflict: 'stripe_subscription_id' });
```

Also: when creating an auto-renew subscription, expire any active manual subscriptions first to prevent dual-active-sub state.

### PITFALL-4: Credit Idempotency — Upgrade vs Renewal

**Problem**: Simple idempotency on `period_start` alone fails when a user upgrades mid-period (same `period_start`, different tier). Credits aren't granted for the upgrade.

**Solution**: Composite idempotency key = `(period_start + tier)`:
```sql
IF v_sub.last_grant_period_start = p_period_start
   AND v_sub.last_grant_tier = v_real_tier THEN
  RETURN false;  -- already granted for this period+tier combo
END IF;
```

Behavior:
- Same period + same tier = skip (duplicate)
- Same period + different tier (upgrade) = accumulate credits
- Different period = reset credits (new billing cycle)

### PITFALL-5: Webhook Version ≠ SDK Version

**Problem**: You configure the Stripe SDK with `apiVersion: '2024-12-18.acacia'`, but webhook events use your **Stripe account's default API version** (set in Dashboard). The event payload format may differ from what the SDK expects.

**Solution**: Always treat webhook event.data.object as `Record<string, any>` and handle both old and new field formats. Don't rely on the SDK's TypeScript types for webhook data.

### PITFALL-6: constructEvent vs constructEventAsync (Deno)

**Problem**: Stripe's Node.js SDK `constructEvent()` uses Node's `crypto.createHmac()` which isn't available in Deno. The async version `constructEventAsync()` may also fail depending on the runtime.

**Solution**: Manual HMAC-SHA256 verification using WebCrypto API (works in all Deno runtimes):
```typescript
const key = await crypto.subtle.importKey(
  'raw', encoder.encode(secret),
  { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']
);
const sig = await crypto.subtle.sign('HMAC', key, encoder.encode(signedPayload));
```

**Critical**: Always verify against the **raw body text**, NOT the parsed JSON. The signature is computed over the raw bytes.

### PITFALL-7: Manual Sub Doesn't Auto-Expire

**Problem**: Manual subscriptions (WeChat/Alipay) have no Stripe-side subscription object, so there's no `customer.subscription.deleted` webhook when the period ends. The subscription stays "ACTIVE" forever in the database.

**Solution**: pg_cron job to expire stale manual subscriptions + downgrade tier:
```sql
SELECT cron.schedule('expire_manual_subs', '0 * * * *',
$$
  WITH expired AS (
    UPDATE pay_subscription
    SET status = 'EXPIRED', updated_at = now()
    WHERE auto_renew = false AND status = 'ACTIVE'
      AND current_period_end < now()
    RETURNING user_id
  )
  UPDATE member_profile SET tier = 'free', updated_at = now()
  WHERE user_id IN (SELECT user_id FROM expired);
$$);
```

### PITFALL-8: Wallet Split Cascading Changes

**Problem**: Adding `topup_balance` to the wallet affects **every RPC function** that touches the wallet. Missing even one function causes incorrect `balance_after` values in transaction logs.

**Affected functions** (all must be updated simultaneously):
1. `billing_deduct_credits` — dual-bucket deduction logic
2. `billing_refund_credits` — refund target (always to `balance`)
3. `billing_grant_subscription_credits` — only reset `balance`, preserve `topup_balance`
4. `billing_activate_manual_subscription` — only set `balance`
5. `billing_renew_manual_subscription` — only set `balance`
6. `billing_admin_adjust_credits` — operate on `topup_balance`
7. `billing_topup_credits` — add to `topup_balance`

**Rule**: `balance_after` in tx logs must always equal `balance + topup_balance`.

---

## Idempotency Patterns Reference

| Operation | Idempotency Key | Check Method |
|-----------|----------------|--------------|
| Webhook events | `stripe_event_id` | UNIQUE constraint on `stripe_webhook_events` |
| Credit grants (renewal) | `period_start + tier` | `last_grant_period_start + last_grant_tier` on `pay_subscription` |
| Credit grants (upgrade) | `period_start + new_tier` | Same composite key — different tier = new grant |
| Refunds | `biz_id + USAGE_REFUND` | Check `billing_wallet_tx` for existing USAGE_REFUND |
| Top-ups | `checkout_session_id + RECHARGE` | Check `billing_wallet_tx` for existing RECHARGE |
| Manual sub renewal | `renew_ + checkout_session_id` | Check `billing_wallet_tx` for existing biz_id |

---

## Debugging Quick Reference

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Fast 401 (< 100ms) | Gateway JWT rejection (ES256/HS256 mismatch) | Deploy with `--no-verify-jwt` |
| Slow 401 (> 500ms) | Function-level `getUser()` failure | Check token format, auth config |
| Webhook 400 | Signature mismatch | Verify using raw body, not parsed JSON |
| Webhook 200 but no effect | Event deduplication blocked it | Check `stripe_webhook_events` table |
| Credits not granted | Idempotency check blocked it | Check `last_grant_period_start + last_grant_tier` |
| Duplicate subscriptions | INSERT instead of UPSERT | Use UPSERT on `stripe_subscription_id` |
| Manual sub never expires | No auto-expiry mechanism | Add pg_cron job |
| `balance_after` wrong | Wallet split not applied to all RPCs | Update ALL 7 wallet-touching functions |
| CORS error | Origin not in allowlist | Check `ALLOWED_ORIGIN` env var, handle www variant |
| Checkout success but no credits | Webhook not configured or failing | Check Stripe Dashboard webhook logs |

---

## Testing & QA

### Stripe Test Mode Setup

1. **API Keys**: Use `sk_test_*` / `pk_test_*` from [Stripe Dashboard > Developers > API Keys](https://dashboard.stripe.com/test/apikeys)
2. **Test Cards**:
   - Success: `4242 4242 4242 4242` (any future exp, any CVC)
   - Decline: `4000 0000 0000 0002`
   - 3D Secure: `4000 0025 0000 3155`
   - Insufficient funds: `4000 0000 0000 9995`

### Stripe CLI for Local Webhook Testing

```bash
# Install
brew install stripe/stripe-cli/stripe

# Login (one-time)
stripe login

# Forward webhooks to local Edge Function (or Next.js endpoint)
stripe listen --forward-to http://localhost:54321/functions/v1/pay-stripe-webhook

# The CLI prints a webhook signing secret (whsec_...) — set it as STRIPE_WEBHOOK_SECRET

# Trigger specific events for testing
stripe trigger checkout.session.completed
stripe trigger customer.subscription.updated
stripe trigger invoice.paid
stripe trigger customer.subscription.deleted
```

### Test Scenarios Checklist

**Subscription Flow**:
- [ ] New user signs up → gets 30 free credits via trigger
- [ ] Free user → Pro subscription checkout → credits granted
- [ ] Pro → Premium upgrade (same period) → additional credits, not reset
- [ ] Premium → Pro downgrade attempt → blocked (3-layer defense)
- [ ] Active sub → cancel → status changes to `canceled` (remains active until period end)
- [ ] Canceled sub → resume → status back to `active`
- [ ] Period renewal → `invoice.paid` → credits reset to new tier amount

**One-Time Top-up Flow**:
- [ ] Top-up checkout → `checkout.session.completed` → `topup_balance` increased
- [ ] Duplicate webhook → idempotency blocks double credit

**Credit Deduction**:
- [ ] Generation with sufficient credits → `balance` deducted first, then `topup_balance`
- [ ] Generation with insufficient total → returns error, no partial deduction
- [ ] Concurrent deductions → `FOR UPDATE` lock prevents race condition

**Manual Subscription (if applicable)**:
- [ ] Manual sub activation → credits granted to `balance`
- [ ] Manual sub expiry → pg_cron sets `status = 'expired'`
- [ ] Manual sub renewal → new checkout, fresh credits

**Edge Cases**:
- [ ] Webhook arrives before checkout redirect completes → user sees credits on refresh
- [ ] Stripe API version mismatch → `extractPeriod()` handles both formats
- [ ] Edge Function cold start → rate limiting still works (DB-backed)

### Stripe Test Clocks (Time Travel Testing)

For subscription lifecycle testing without waiting for real billing cycles:

```bash
# Create a test clock via Stripe Dashboard:
# Stripe Dashboard > Developers > Test Clocks > New

# Create a customer attached to the clock
# Subscribe the customer to a plan
# Advance the clock to simulate:
#   - End of trial
#   - First renewal
#   - Payment failure + retry cycle
#   - Subscription cancellation at period end
```

Test Clocks let you verify that `invoice.paid`, `customer.subscription.updated`, and expiry events fire correctly without waiting 30 days.

### Idempotency Regression Tests

Replay each webhook event type twice and verify:
1. First call: processes normally, returns 200
2. Second call (same `event.id`): skipped by `stripe_webhook_events` dedup, returns 200
3. Database state unchanged after second call

```bash
# Using Stripe CLI to replay events:
stripe events resend evt_xxxxxxxxxxxxx
```

---

## Deployment Checklist

### Supabase
- [ ] All migrations applied in order
- [ ] RLS enabled on all tables
- [ ] `billing_handle_new_user` trigger exists on `auth.users`
- [ ] pg_cron extension enabled (for cleanup + manual sub expiry)
- [ ] All Edge Functions deployed with `--no-verify-jwt`

### Stripe
- [ ] Webhook endpoint configured with correct URL
- [ ] Events subscribed: `checkout.session.completed`, `customer.subscription.created`, `customer.subscription.updated`, `invoice.paid`, `customer.subscription.deleted`
- [ ] Webhook signing secret set as `STRIPE_WEBHOOK_SECRET` env var
- [ ] Account API version noted (may differ from SDK version)

### Edge Function Environment Variables
- [ ] `SUPABASE_URL`
- [ ] `SUPABASE_SERVICE_ROLE_KEY`
- [ ] `STRIPE_SECRET_KEY`
- [ ] `STRIPE_WEBHOOK_SECRET`
- [ ] `APP_BASE_URL` (e.g., `https://your-app.com`)
- [ ] `ALLOWED_ORIGIN` (CORS origin, or `*` for dev)

### Frontend
- [ ] `NEXT_PUBLIC_SUPABASE_URL` set
- [ ] `NEXT_PUBLIC_SUPABASE_ANON_KEY` set
- [ ] Pricing constants match Stripe Dashboard price IDs
- [ ] Return URLs use relative paths (built server-side)

---

## Key Design Decisions

1. **Credit-based billing** over per-request pricing: simpler UX, predictable costs
2. **Dual wallet** over single balance: subscription credits reset monthly, purchased credits persist
3. **Database RPC** over application-level logic: atomic transactions, no race conditions
4. **`FOR UPDATE` row locks**: prevents concurrent deduction/grant conflicts
5. **Webhook-driven state sync**: Stripe is source of truth for auto-renew subscriptions
6. **DB-backed rate limiting**: survives Deno cold starts and multi-isolate deployments
7. **3-layer downgrade prevention**: defense in depth (frontend, edge function, database)
8. **Manual HMAC verification**: works in all Deno runtimes (no Node.js crypto dependency)

---

## Monitoring & Observability

### Edge Function Logging

Supabase Edge Functions log to the Supabase Dashboard (Logs > Edge Functions). Structure your logs for searchability:

```typescript
// In Edge Function handlers — use structured JSON logging
console.log(JSON.stringify({
  event: 'webhook_processed',
  stripe_event_id: event.id,
  type: event.type,
  user_id: userId,
  credits_granted: amount,
  duration_ms: Date.now() - startTime,
}))

// Error logging with context
console.error(JSON.stringify({
  event: 'webhook_error',
  stripe_event_id: event.id,
  error: error.message,
  stack: error.stack,
}))
```

### Stripe Webhook Health

Monitor webhook delivery health in the Stripe Dashboard:

1. **Stripe Dashboard > Developers > Webhooks > [your endpoint]**
   - Check delivery success rate (target: > 99%)
   - Review failed events and their error responses
   - Re-send failed events after fixing the handler

2. **Key Webhook Metrics to Watch**:
   - Pending event queue (should stay near 0)
   - Average response time (should be < 5s, ideally < 1s)
   - Failed delivery count (should be 0 in steady state)

3. **Stripe Auto-Retry Schedule**: Events that fail are retried over 3 days with exponential backoff. If your endpoint is down temporarily, events will eventually arrive.

### Key Billing Metrics (SQL Queries)

```sql
-- Active subscriptions by tier
SELECT tier, status, count(*)
FROM pay_subscription
GROUP BY tier, status;

-- Daily revenue (from wallet transactions)
SELECT date_trunc('day', created_at) AS day,
       sum(amount) AS total_credits_deducted
FROM billing_wallet_tx
WHERE tx_type = 'USAGE'
GROUP BY day ORDER BY day DESC LIMIT 30;

-- Users with low balance (potential churn signals)
SELECT w.user_id, w.balance, w.topup_balance,
       p.display_name, p.tier
FROM billing_wallet w
JOIN member_profile p ON p.id = w.user_id
WHERE w.balance + w.topup_balance < 10
  AND p.tier != 'free'
ORDER BY w.balance + w.topup_balance;

-- Webhook processing lag
SELECT event_type,
       avg(extract(epoch from processed_at - created_at)) AS avg_lag_seconds
FROM stripe_webhook_events
WHERE created_at > now() - interval '7 days'
GROUP BY event_type;

-- Failed webhook events (unprocessed)
SELECT * FROM stripe_webhook_events
WHERE processed_at IS NULL
  AND created_at > now() - interval '3 days'
ORDER BY created_at DESC;
```

### Alerting Strategy

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| Webhook queue growing | Pending events > 10 | HIGH | Check Edge Function logs, redeploy if needed |
| Credit balance mismatch | `balance_after != balance + topup_balance` | CRITICAL | Run reconciliation query, check RPC logic |
| Subscription without wallet | `pay_subscription` exists but no `billing_wallet` | CRITICAL | Check `billing_handle_new_user` trigger |
| High webhook latency | Avg response > 5s | MEDIUM | Check for slow DB queries, cold starts |
| Duplicate subscription rows | Multiple active subs per user | HIGH | Check UPSERT logic in webhook handler |

### Production Health Check Query

Run this periodically to verify system integrity:

```sql
-- Integrity check: every subscription user has a wallet
SELECT s.user_id, s.tier, s.status
FROM pay_subscription s
LEFT JOIN billing_wallet w ON w.user_id = s.user_id
WHERE w.user_id IS NULL AND s.status = 'active';
-- Should return 0 rows

-- Integrity check: balance_after consistency
SELECT t.id, t.balance_after,
       w.balance + w.topup_balance AS actual_total
FROM billing_wallet_tx t
JOIN billing_wallet w ON w.user_id = t.user_id
WHERE t.id = (
  SELECT max(id) FROM billing_wallet_tx t2
  WHERE t2.user_id = t.user_id
)
AND t.balance_after != w.balance + w.topup_balance;
-- Should return 0 rows
```

---

## Related Skills

| Skill | When to Use |
|-------|-------------|
| **mvp-billing-system-cn** | Chinese version of this skill (same content, 中文) |
| **stripe-payments** | Lighter intro to Stripe integration (Next.js Route Handlers, no credits/wallets) |
| **supabase-developer** | Full Supabase reference (Auth, Database, Storage, Real-time) |
| **supabase-gemini-deploy** | Edge Function deployment troubleshooting (JWT, CORS, 401/500 diagnosis) |
| **deploy-gate** | Pre-deployment gate checks (6 gates including build, env vars, git) |
| **nextjs-fullstack** | Next.js App Router patterns (Server Components, Actions, Middleware) |
| **saas-quickstart** | Quick start from starter kit (5 steps to launch) |
| **project-scaffold** | Full architecture document generation + constraint code skeleton |
