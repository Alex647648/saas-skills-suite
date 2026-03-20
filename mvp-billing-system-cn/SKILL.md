---
name: mvp-billing-system-cn
description: |
  完整的SaaS计费系统实施手册 —— 从数据库设计到生产部署。
  覆盖 Supabase + Stripe + Next.js 双轨支付（订阅 + 一次性）、
  双钱包（订阅积分 + 加购积分）、积分计费、Webhook调试，
  以及8个实战踩坑及解决方案。
  触发关键词：「计费系统」「支付集成」「Stripe接入」
  「订阅管理」「积分系统」「webhook调试」
  「SaaS支付」「收银流程」「定价页面」「支付系统」
---

# MVP 计费系统实施手册 —— 从零到生产

> 基于 4 天 32 次提交的实战经验蒸馏而成。
> 技术栈：Next.js 15 + Supabase + Stripe

## 适用场景

- 从零构建 SaaS 产品的订阅 + 积分计费系统
- 将 Stripe 与 Supabase Edge Functions 集成
- 实现双轨支付（银行卡自动续费 + 微信/支付宝一次性付款）
- 设计带幂等保障的积分钱包系统
- 调试 Webhook 故障或支付数据不一致

## 架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                       前端 (Next.js)                            │
│  PricingPanel → PaymentMethodModal → billingService.ts          │
│  SubscriptionCard / PaywallModal / CreditBadge / BillingHistory │
└──────────┬──────────────────────────────────┬───────────────────┘
           │ Bearer token                     │ Bearer token
           ▼                                  ▼
┌─────────────────────┐          ┌────────────────────────────────┐
│  pay-create-checkout │          │  pay-manage-subscription       │
│  (3条分支:           │          │  (取消/恢复/切换/管理门户)      │
│   subscription,      │          └────────────────────────────────┘
│   one_time, topup)   │
└──────────┬───────────┘
           │ Stripe Checkout URL
           ▼
┌─────────────────────┐    webhook     ┌──────────────────────────┐
│   Stripe 收银台      │──────────────→│  pay-stripe-webhook       │
│   (托管页面)         │               │  (签名验证 + 事件去重      │
└──────────────────────┘               │   + 多事件分发)           │
                                       └──────────┬───────────────┘
                                                  │ RPC 调用
                                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                     SUPABASE (PostgreSQL)                        │
│  member_profile ←→ billing_wallet ←→ billing_wallet_tx          │
│  pay_subscription    pay_checkout_session                        │
│  stripe_webhook_events   rate_limit_log   admin_audit_log       │
│                                                                  │
│  RPC函数: billing_deduct_credits, billing_refund_credits,       │
│          billing_grant_subscription_credits,                     │
│          billing_activate_manual_subscription,                   │
│          billing_renew_manual_subscription,                      │
│          billing_topup_credits, check_rate_limit                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 第一阶段：数据库表设计（第1天）

### 核心表

设计原则：profile 和 wallet 与用户 **1:1 关系**，流水表 **只追加不修改**。

#### 1. member_profile — 会员计费档案

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

#### 2. billing_wallet — 积分钱包（双桶设计）

```sql
CREATE TABLE billing_wallet (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL UNIQUE REFERENCES auth.users(id) ON DELETE CASCADE,
  balance         NUMERIC(10,1) NOT NULL DEFAULT 0     -- 订阅积分（每月重置）
                  CHECK (balance >= 0),
  topup_balance   NUMERIC(10,1) NOT NULL DEFAULT 0     -- 加购积分（永不过期）
                  CHECK (topup_balance >= 0),
  freeze_amount   NUMERIC(10,1) NOT NULL DEFAULT 0
                  CHECK (freeze_amount >= 0),
  total_consumed  NUMERIC(10,1) NOT NULL DEFAULT 0,
  total_recharged NUMERIC(10,1) NOT NULL DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

核心设计：**双桶钱包**
- `balance` = 订阅积分，每个计费周期重置
- `topup_balance` = 加购积分，永不过期
- 扣减优先级：先扣 `balance`（即将过期），再扣 `topup_balance`

#### 3. billing_wallet_tx — 流水账本（只追加）

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
  biz_id        TEXT,                    -- 外部引用（用于幂等检查）
  title         TEXT NOT NULL,
  amount        NUMERIC(10,1) NOT NULL,  -- 正数=入账，负数=扣减
  balance_after NUMERIC(10,1) NOT NULL CHECK (balance_after >= 0),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wallet_tx_user    ON billing_wallet_tx(user_id, created_at DESC);
CREATE INDEX idx_wallet_tx_biztype ON billing_wallet_tx(biz_type, created_at DESC);
```

#### 4. pay_subscription — 订阅记录

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
  auto_renew              BOOLEAN NOT NULL DEFAULT true,  -- false = 手动续费（微信/支付宝）
  last_grant_period_start TIMESTAMPTZ,    -- 幂等：上次发放积分的周期起点
  last_grant_tier         TEXT,           -- 幂等：上次发放积分的套餐等级
  created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 防止同一用户同时拥有多个活跃的手动订阅
CREATE UNIQUE INDEX idx_unique_active_manual_sub
  ON pay_subscription (user_id)
  WHERE auto_renew = false AND status IN ('ACTIVE', 'PAST_DUE');
```

#### 5. stripe_webhook_events — Webhook 去重表

```sql
CREATE TABLE stripe_webhook_events (
  id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  stripe_event_id TEXT NOT NULL UNIQUE,
  event_type      TEXT NOT NULL,
  processed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 新用户触发器

注册时自动创建 profile + wallet 并赠送初始积分：

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

### RLS 策略

```sql
-- 用户只能查看自己的数据
ALTER TABLE member_profile ENABLE ROW LEVEL SECURITY;
CREATE POLICY "用户读取自己的档案"
  ON member_profile FOR SELECT USING (auth.uid() = user_id);

ALTER TABLE billing_wallet ENABLE ROW LEVEL SECURITY;
CREATE POLICY "用户读取自己的钱包"
  ON billing_wallet FOR SELECT USING (auth.uid() = user_id);

ALTER TABLE billing_wallet_tx ENABLE ROW LEVEL SECURITY;
CREATE POLICY "用户读取自己的流水"
  ON billing_wallet_tx FOR SELECT USING (auth.uid() = user_id);

ALTER TABLE pay_subscription ENABLE ROW LEVEL SECURITY;
CREATE POLICY "用户读取自己的订阅"
  ON pay_subscription FOR SELECT USING (auth.uid() = user_id);
```

---

## 第二阶段：计费 RPC 函数（第1天）

所有 RPC 使用 `SECURITY DEFINER` + `SET search_path = public` 保障权限提升安全。

### 流水号生成器

```sql
CREATE OR REPLACE FUNCTION billing_generate_tx_no()
RETURNS TEXT LANGUAGE plpgsql AS $$
BEGIN
  RETURN 'WT-' || to_char(now(), 'YYYYMMDDHH24MISS')
       || '-' || upper(substr(replace(gen_random_uuid()::text, '-', ''), 1, 8));
END;
$$;
```

### billing_deduct_credits — 原子化双桶扣减

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
  -- 输入校验
  IF p_amount <= 0 THEN
    RETURN QUERY SELECT false, 0::NUMERIC(10,1), 0, 'INVALID_AMOUNT'::TEXT; RETURN;
  END IF;

  -- 加锁防并发（profile + wallet 同时锁定）
  SELECT * INTO v_profile FROM member_profile WHERE user_id = p_user_id FOR UPDATE;
  IF NOT FOUND THEN
    RETURN QUERY SELECT false, 0::NUMERIC(10,1), 0, 'USER_NOT_FOUND'::TEXT; RETURN;
  END IF;

  SELECT * INTO v_wallet FROM billing_wallet WHERE user_id = p_user_id FOR UPDATE;

  -- 检查总可用额度
  v_total := v_wallet.balance + v_wallet.topup_balance;
  IF v_total < p_amount THEN
    RETURN QUERY SELECT false, v_total, 0, 'INSUFFICIENT_CREDITS'::TEXT; RETURN;
  END IF;

  -- 扣减优先级：先扣 balance（即将过期），再扣 topup_balance
  IF v_wallet.balance >= p_amount THEN
    v_from_balance := p_amount;  v_from_topup := 0;
  ELSE
    v_from_balance := v_wallet.balance;
    v_from_topup   := p_amount - v_wallet.balance;
  END IF;

  v_new_balance := v_wallet.balance - v_from_balance;
  v_new_topup   := v_wallet.topup_balance - v_from_topup;

  -- 乐观锁更新
  UPDATE billing_wallet
    SET balance = v_new_balance, topup_balance = v_new_topup,
        total_consumed = total_consumed + p_amount, updated_at = now()
    WHERE id = v_wallet.id
      AND balance >= v_from_balance AND topup_balance >= v_from_topup;

  GET DIAGNOSTICS v_updated = ROW_COUNT;
  IF v_updated = 0 THEN
    RETURN QUERY SELECT false, v_total, 0, 'INSUFFICIENT_CREDITS'::TEXT; RETURN;
  END IF;

  -- 写入流水
  v_tx_no := billing_generate_tx_no();
  INSERT INTO billing_wallet_tx (tx_no, wallet_id, user_id, biz_type, biz_id, title, amount, balance_after)
    VALUES (v_tx_no, v_wallet.id, p_user_id, 'USAGE', p_biz_id,
            LEFT(p_action, 50) || ' · ' || LEFT(p_model, 50),
            -p_amount, v_new_balance + v_new_topup);

  RETURN QUERY SELECT true, (v_new_balance + v_new_topup), 0, NULL::TEXT;
END;
$$;
```

### billing_refund_credits — 幂等退款

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

  -- 幂等检查：已退过的不再退
  IF EXISTS (SELECT 1 FROM billing_wallet_tx
    WHERE biz_id = p_biz_id AND biz_type = 'USAGE_REFUND' AND user_id = p_user_id
  ) THEN RETURN; END IF;

  -- 验证存在对应的扣减记录（拒绝空退）
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
            '退款: ' || LEFT(p_reason, 200), p_amount, v_new_bal + v_wallet.topup_balance);
END;
$$;
```

### billing_grant_subscription_credits — 复合幂等积分发放

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
  -- 锁定订阅行
  SELECT * INTO v_sub FROM pay_subscription
    WHERE user_id = p_user_id AND status = 'ACTIVE' FOR UPDATE;
  IF NOT FOUND THEN RETURN false; END IF;

  -- 从锁定的行中获取套餐（永远不信任调用方传入的 tier）
  v_real_tier := v_sub.tier;

  -- 复合幂等：相同周期 + 相同套餐 = 已发放
  IF v_sub.last_grant_period_start IS NOT NULL
     AND v_sub.last_grant_period_start = p_period_start
     AND v_sub.last_grant_tier = v_real_tier THEN
    RETURN false;
  END IF;

  -- 确定发放额度
  v_grant := CASE v_real_tier
    WHEN 'basic' THEN 1000 WHEN 'pro' THEN 5000 WHEN 'max' THEN 20000 ELSE 0
  END;
  IF v_grant = 0 THEN RETURN false; END IF;

  -- 更新钱包
  -- 同周期（升级）→ 累加 | 新周期（续费）→ 重置
  SELECT * INTO v_wallet FROM billing_wallet WHERE user_id = p_user_id FOR UPDATE;
  IF v_sub.last_grant_period_start = p_period_start THEN
    v_new_bal := v_wallet.balance + v_grant;   -- 累加（升级）
  ELSE
    v_new_bal := v_grant;                      -- 重置（新周期）
  END IF;

  UPDATE billing_wallet
    SET balance = v_new_bal, total_recharged = total_recharged + v_grant, updated_at = now()
    WHERE id = v_wallet.id;

  -- 标记发放周期和套餐（复合幂等键）
  UPDATE pay_subscription
    SET last_grant_period_start = p_period_start, last_grant_tier = v_real_tier, updated_at = now()
    WHERE id = v_sub.id;

  UPDATE member_profile SET tier = v_real_tier, updated_at = now() WHERE user_id = p_user_id;

  v_tx_no := billing_generate_tx_no();
  INSERT INTO billing_wallet_tx (tx_no, wallet_id, user_id, biz_type, biz_id, title, amount, balance_after)
    VALUES (v_tx_no, v_wallet.id, p_user_id, 'SUBSCRIPTION_GRANT', v_sub.id::TEXT,
            v_real_tier || ' 月度积分', v_grant, v_new_bal + v_wallet.topup_balance);
  RETURN true;
END;
$$;
```

### billing_topup_credits — 加购积分

```sql
CREATE OR REPLACE FUNCTION billing_topup_credits(
  p_user_id UUID, p_amount NUMERIC(10,1), p_checkout_session_id TEXT
)
RETURNS BOOLEAN LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, extensions
AS $$
DECLARE v_wallet billing_wallet%ROWTYPE; v_tx_no TEXT; v_new_topup NUMERIC(10,1);
BEGIN
  IF p_amount <= 0 THEN RAISE EXCEPTION 'INVALID_AMOUNT'; END IF;

  -- 幂等：跳过已处理的 checkout session
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
            '积分加购 +' || p_amount::TEXT, p_amount, v_wallet.balance + v_new_topup);
  RETURN true;
END;
$$;
```

---

## 第三阶段：Edge Functions — Stripe 集成（第2天）

### 共享模块模式 (`_shared/`)

#### `_shared/cors.ts` — CORS + JSON 响应辅助

```typescript
export function corsHeaders(req?: Request): Record<string, string> {
  const origin = req?.headers.get('origin') ?? '';
  const allowed = Deno.env.get('ALLOWED_ORIGIN') ?? 'https://your-app.com';
  // 同时支持 www 和非 www 变体
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

#### `_shared/pricing.ts` — 价格常量单一数据源

```typescript
// Stripe price ID → 内部套餐/周期映射
export const PRICE_TO_TIER: Record<string, string> = {
  'price_basic_monthly':  'basic',   // 替换为实际的 Stripe price ID
  'price_basic_yearly':   'basic',
  'price_pro_monthly':    'pro',
  'price_pro_yearly':     'pro',
  'price_max_monthly':    'max',
  'price_max_yearly':     'max',
};

export const PRICE_TO_INTERVAL: Record<string, string> = { /* 同上映射 */ };
export const ALLOWED_PRICE_IDS: string[] = Object.keys(PRICE_TO_TIER);

// Webhook 金额交叉验证用的期望金额（最小货币单位）
export const EXPECTED_AMOUNTS: Record<string, Record<string, number>> = {
  basic: { month: 2900, year: 24900 },
  pro:   { month: 7900, year: 69900 },
  max:   { month: 19900, year: 179900 },
};

// 加购积分包
export const TOPUP_PACKS: Record<string, { credits: number; amount: number; name: string }> = {
  small:  { credits: 800,  amount: 2900,  name: '小包 · 800积分' },
  medium: { credits: 4000, amount: 7900,  name: '中包 · 4000积分' },
  large:  { credits: 8000, amount: 10000, name: '大包 · 8000积分' },
};

// Stripe 状态 → 内部状态映射
export const STRIPE_STATUS_MAP: Record<string, string> = {
  active: 'ACTIVE', trialing: 'ACTIVE',
  past_due: 'PAST_DUE', unpaid: 'PAST_DUE', incomplete: 'PAST_DUE',
  incomplete_expired: 'EXPIRED', canceled: 'CANCELED', paused: 'PAST_DUE',
};
```

### pay-create-checkout — 三路收银

```typescript
// Edge Function: pay-create-checkout/index.ts
// 处理 3 种支付流程: subscription, one_time, topup

Deno.serve(async (req: Request) => {
  // 1. CORS 预检
  if (req.method === 'OPTIONS') return new Response('ok', { headers: corsHeaders(req) });

  // 2. 载荷大小限制
  const contentLength = parseInt(req.headers.get('content-length') ?? '0', 10);
  if (contentLength > 65_536) return jsonResponse({ ok: false, error_code: 'PAYLOAD_TOO_LARGE' }, 413, req);

  // 3. 认证：使用 getUser(token)，不依赖网关 JWT 验证
  const token = req.headers.get('Authorization')?.replace('Bearer ', '');
  const { data: { user } } = await supabase.auth.getUser(token);

  // 4. 数据库限流
  const { data: rateLimitOk } = await supabase.rpc('check_rate_limit', {
    p_user_id: user.id, p_action: 'checkout', p_max: 10, p_window_seconds: 60,
  });

  // 5. 解析请求并校验
  const { price_id, return_path, payment_flow, is_renewal, topup_pack_id } = await req.json();

  // 6. 阻止重复订阅（相同套餐 + 周期）
  // 7. 阻止手动订阅降级

  // 8. 获取或创建 Stripe 客户
  // 9. 服务端构建回调 URL（绝不从客户端 origin 构建）
  const appBaseUrl = Deno.env.get('APP_BASE_URL');

  // 10. 按 payment_flow 分支：
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
      // metadata 要同时写在 session 和 subscription_data 上
  }

  // 11. 记录到 pay_checkout_session 表
  return jsonResponse({ ok: true, checkout_url: session.url }, 200, req);
});
```

### pay-stripe-webhook — 健壮的事件处理

```typescript
// 关键：Deno 中手动 HMAC-SHA256 签名验证
async function verifyStripeSignature(payload: string, sigHeader: string, secret: string): Promise<boolean> {
  const parts = sigHeader.split(',').reduce<Record<string, string>>((acc, part) => {
    const [k, v] = part.split('=');
    acc[k] = v;
    return acc;
  }, {});
  const timestamp = parts['t'];
  const signature = parts['v1'];
  if (!timestamp || !signature) return false;

  // 拒绝超过5分钟的事件
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

// 处理 Stripe API 版本变更的周期字段
function extractPeriod(rawSub: Record<string, any>): { periodStart: string; periodEnd: string } {
  const cp = rawSub.current_period;
  const startRaw = cp?.start ?? rawSub.current_period_start;  // 新 vs 旧 API
  const endRaw = cp?.end ?? rawSub.current_period_end;
  return { periodStart: toISO(startRaw), periodEnd: toISO(endRaw) };
}

// 事件处理流程：
// 1. 验证签名（原始 body，不是解析后的 JSON）
// 2. 通过 stripe_webhook_events 表去重
// 3. 先 INSERT 声明占有事件（claim）
// 4. 按 event.type 分发：
//    - checkout.session.completed → 处理 topup / one_time / subscription
//    - customer.subscription.created/updated → upsert 订阅 + 发放积分
//    - invoice.paid → 续费积分发放
//    - customer.subscription.deleted → 过期订阅 + 降级套餐
// 5. 处理错误时返回 500（Stripe 会重试最多 3 天）
```

### pay-manage-subscription — 4 种操作

```typescript
// 操作: cancel, resume, change_plan, portal
// 核心模式: 判断订阅是否为手动续费 (auto_renew === false)
//   手动订阅: 直接操作数据库（不需要调 Stripe API）
//   自动续费: 调 Stripe API 后同步数据库

switch (action) {
  case 'cancel':
    // 手动: 直接将状态设为 'CANCELED'
    // 自动: stripe.subscriptions.update({ cancel_at_period_end: true })
    break;
  case 'resume':
    // 手动: 仅在有效期内可恢复
    // 自动: stripe.subscriptions.update({ cancel_at_period_end: false })
    break;
  case 'change_plan':
    // 手动: 不支持（需等过期后重新订阅）
    // 自动: stripe.subscriptions.update({ items: [{ id, price: new_price_id }] })
    break;
  case 'portal':
    // 手动: 不支持
    // 自动: stripe.billingPortal.sessions.create()
    break;
}
```

---

## 第四阶段：前端计费组件（第2天）

### 类型定义

```typescript
export type BillingTier = 'free' | 'basic' | 'pro' | 'max';
export type BillingInterval = 'month' | 'year';
export type SubscriptionStatus = 'ACTIVE' | 'PAST_DUE' | 'CANCELED' | 'EXPIRED';
export type PaymentFlow = 'subscription' | 'one_time' | 'topup';

export interface PlanOption {
  readonly tier: Exclude<BillingTier, 'free'>;
  readonly name: string;
  readonly monthlyPrice: number;     // 最小货币单位（分）
  readonly yearlyPrice: number;
  readonly monthlyCredits: number;
  readonly models: string[];
  readonly stripePriceIds: { readonly monthly: string; readonly yearly: string };
}
```

### billingService — 前端 API 服务层

```typescript
const FUNCTION_URL = process.env.NEXT_PUBLIC_SUPABASE_URL + '/functions/v1';

async function invokeFunction<T>(fnName: string, body: Record<string, unknown>): Promise<T> {
  const { data: { session } } = await supabase.auth.getSession();
  if (!session) throw new Error('请先登录');

  const res = await fetch(`${FUNCTION_URL}/${fnName}`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${session.access_token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(body),
  });
  const result = await res.json();
  if (!res.ok || result.ok === false) throw new Error(result.message || `请求失败: ${res.status}`);
  return result as T;
}

export const billingService = {
  getProfile:       () => supabase.from('member_profile').select('*').single(),
  getWallet:        () => supabase.from('billing_wallet').select('*').single(),
  getTransactions:  (limit, offset) => supabase.from('billing_wallet_tx').select('*').order(...).range(...),
  getActiveSubscription: () => supabase.from('pay_subscription').select('*').in('status', [...]).order(...).limit(1).maybeSingle(),
  createCheckout:   (priceId, flow, isRenewal) => invokeFunction('pay-create-checkout', { price_id, return_path, payment_flow, is_renewal }),
  cancelSubscription: () => invokeFunction('pay-manage-subscription', { action: 'cancel' }),
  resumeSubscription: () => invokeFunction('pay-manage-subscription', { action: 'resume' }),
  changePlan:       (newPriceId) => invokeFunction('pay-manage-subscription', { action: 'change_plan', new_price_id }),
  topUpCredits:     (packId) => invokeFunction('pay-create-checkout', { return_path, payment_flow: 'topup', topup_pack_id }),
  getPortalUrl:     () => invokeFunction('pay-manage-subscription', { action: 'portal' }),
} as const;
```

### 组件层级

```
AccountDashboard（账户仪表盘）
├── CreditBadge          — 实时积分显示（balance + topup_balance）
├── SubscriptionCard     — 订阅状态 + 管理操作（取消/恢复/续费）
├── PricingPanel         — 套餐选择弹窗
│   └── PaymentMethodModal — 银行卡 vs 微信/支付宝
├── PaywallModal         — 积分不足时触发
│   └── TopUpOptions     — 快捷积分加购包
├── BillingHistory       — 流水记录 + 支付记录
└── ArticleArchiveList   — 已生成文章历史
```

### PricingPanel — 核心交互逻辑

```typescript
// 1. 升降级判断
const handleSubscribe = (plan: PlanOption) => {
  const currentRank = TIER_RANK[currentTier];
  const planRank = TIER_RANK[plan.tier];

  // 手动订阅不允许降级
  if (isManualSub && planRank < currentRank) return;
  // 相同套餐+周期不允许重复购买
  if (isManualSub && planRank === currentRank && interval === currentInterval) return;

  if (currentTier !== 'free' && !isManualSub) {
    // 自动续费用户：通过 Stripe change_plan API 切换
    handleChangePlan(plan);
  } else {
    // 免费用户或手动订阅用户：弹出支付方式选择
    setPendingPlan(plan);
  }
};

// 2. 选择支付方式后 → 创建 checkout → 跳转
const handlePaymentMethodSelect = async (flow: PaymentFlow) => {
  const url = await billingService.createCheckout(priceId, flow);
  window.location.href = url;  // 跳转到 Stripe 收银台
};
```

---

## 第五阶段：进阶 — 双轨支付与双桶钱包（第3天）

### 手动订阅设计（微信/支付宝）

对于不支持自动续费的支付方式，使用手动订阅 (`auto_renew: false`)：

```
流程: 用户 → PaymentMethodModal → "微信支付" → one_time checkout
     → Stripe 支付 → webhook checkout.session.completed
     → billing_activate_manual_subscription（数据库创建订阅 + 发放积分）
```

生命周期：**激活 → 取消 → 恢复 → 过期 → 续费**

- 取消：仅设置状态为 'CANCELED'（有效期内仍可使用）
- 恢复：设回 'ACTIVE'（仅在有效期内可操作）
- 过期：pg_cron 定时任务自动处理，或读取时检查
- 续费：新的一次性付款 → billing_renew_manual_subscription

### 钱包拆分：balance vs topup_balance

```
┌─────────────────────────────────┐
│         billing_wallet          │
│  ┌──────────┐  ┌──────────────┐│
│  │ balance  │  │ topup_balance││
│  │(订阅积分 │  │(加购积分，   ││
│  │ 每月重置)│  │ 永不过期)    ││
│  └──────────┘  └──────────────┘│
│       ▲                ▲       │
│       │                │       │
│  订阅积分           加购积分    │
│  发放 RPC         充值 RPC     │
└─────────────────────────────────┘

扣减优先级：balance → topup_balance
退款目标：始终退到 balance（订阅积分桶）
订阅发放：仅重置 balance，topup_balance 不动
```

### 降级防护（三层防御）

```
第一层: 前端 — PricingPanel 禁用降级按钮
第二层: Edge Function — pay-create-checkout 检查 TIER_RANK 后再创建 checkout
第三层: 数据库 — billing_activate_manual_subscription 抛出 DOWNGRADE_BLOCKED 异常
```

```sql
-- 第三层：数据库兜底（最后防线）
SELECT * INTO v_old_sub FROM pay_subscription
  WHERE user_id = p_user_id AND auto_renew = false
    AND status IN ('ACTIVE','PAST_DUE','CANCELED')
    AND current_period_end > now()
  FOR UPDATE;

IF FOUND THEN
  v_old_rank := CASE v_old_sub.tier WHEN 'basic' THEN 1 WHEN 'pro' THEN 2 WHEN 'max' THEN 3 ELSE 0 END;
  v_new_rank := CASE p_tier WHEN 'basic' THEN 1 WHEN 'pro' THEN 2 WHEN 'max' THEN 3 ELSE 0 END;
  IF v_new_rank < v_old_rank THEN
    RAISE EXCEPTION 'DOWNGRADE_BLOCKED: 不能从 % 降级到 %', v_old_sub.tier, p_tier;
  END IF;
END IF;
```

---

## 第六阶段：安全加固与生产化（第4天）

### Webhook 事件去重

```sql
-- 表: stripe_webhook_events（stripe_event_id 唯一约束）
-- 模式: 处理前先 INSERT，唯一约束失败则跳过

const { data: existingEvent } = await supabase
  .from('stripe_webhook_events')
  .select('id').eq('stripe_event_id', event.id).maybeSingle();
if (existingEvent) return; // 已处理

const { error: insertErr } = await supabase.from('stripe_webhook_events').insert({
  stripe_event_id: event.id, event_type: event.type,
});
if (insertErr) return; // 被其他实例抢占
```

### 金额交叉验证

```typescript
// 在 webhook 中：验证 metadata 中的 tier/interval 与 checkout 记录是否一致
const { data: csRecord } = await supabase
  .from('pay_checkout_session')
  .select('stripe_price_id')
  .eq('stripe_session_id', session.id)
  .maybeSingle();

if (csRecord?.stripe_price_id) {
  const expectedTier = PRICE_TO_TIER[csRecord.stripe_price_id];
  if (tier !== expectedTier) {
    console.error('metadata 不匹配!');
    break; // 中止处理
  }
}

// 同时验证实际支付金额
const expectedAmount = EXPECTED_AMOUNTS[tier]?.[billingInterval];
if (expectedAmount && actualAmount < expectedAmount) {
  console.error('支付金额不匹配!');
  break;
}
```

### 数据库限流

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

优势：**抗冷启动 + 多实例共享**（内存限流在 Deno isolate 冷启动时会重置）。

### 管理员审计日志

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
-- 不添加 RLS 策略 = 仅 service_role 可读写
```

### pg_cron 定时清理

```sql
-- 清理 30 天前的 webhook 事件
SELECT cron.schedule('cleanup_webhook_events', '0 3 * * *',
  $$DELETE FROM stripe_webhook_events WHERE processed_at < now() - interval '30 days'$$);

-- 清理 7 天前的限流日志
SELECT cron.schedule('cleanup_rate_limit_log', '5 3 * * *',
  $$DELETE FROM rate_limit_log WHERE created_at < now() - interval '7 days'$$);
```

---

## 踩坑百科全书

### 坑-1：Stripe API 版本地狱

**问题**：Stripe 经常引入破坏性变更。2025年2月版本移除了订阅对象上的 `current_period_start`/`current_period_end`，替换为 `current_period` 对象 `{ start, end }`。

**解决方案**：构建版本无关的提取器：
```typescript
function extractPeriod(rawSub: Record<string, any>) {
  const cp = rawSub.current_period;
  const start = cp?.start ?? rawSub.current_period_start;
  const end = cp?.end ?? rawSub.current_period_end;
  return { periodStart: toISO(start), periodEnd: toISO(end) };
}
```

### 坑-2：Deno Edge Function JWT（ES256 vs HS256）

**问题**：Supabase Auth 用 **ES256**（非对称）签发用户 JWT，但 Edge Function API 网关只验证 **HS256**（对称，用 anon key）。结果：所有用户请求在网关层被 401 拒绝。

**症状**：快速 401 响应（< 100ms）= 网关拒绝。

**解决方案**：部署时跳过网关验证，在函数内验证：
```bash
supabase functions deploy my-function --no-verify-jwt
```
```typescript
// 函数内部验证：
const { data: { user } } = await supabase.auth.getUser(token);
```

### 坑-3：切换套餐导致重复订阅

**问题**：Stripe 切换套餐时触发 `customer.subscription.updated`。如果用 `INSERT` 而不是 `UPSERT`，会创建重复的订阅记录。

**解决方案**：始终对 `stripe_subscription_id` 使用 UPSERT：
```typescript
await supabase.from('pay_subscription')
  .upsert(payload, { onConflict: 'stripe_subscription_id' });
```

另外：创建自动续费订阅时，先过期所有活跃的手动订阅，防止双重活跃状态。

### 坑-4：积分幂等 — 升级 vs 续费

**问题**：仅用 `period_start` 做幂等键，升级时（同一 `period_start`，不同 tier）不会发放新积分。

**解决方案**：复合幂等键 = `(period_start + tier)`：
```sql
IF v_sub.last_grant_period_start = p_period_start
   AND v_sub.last_grant_tier = v_real_tier THEN
  RETURN false;  -- 相同周期+套餐=已发放
END IF;
```

行为：
- 同周期 + 同套餐 = 跳过（重复）
- 同周期 + 不同套餐（升级）= 累加积分
- 不同周期 = 重置积分（新计费周期）

### 坑-5：Webhook 版本 ≠ SDK 版本

**问题**：你在 SDK 中配置 `apiVersion: '2024-12-18.acacia'`，但 webhook 事件使用的是 Stripe **账号默认 API 版本**（在 Dashboard 中设置）。事件载荷格式可能与 SDK 期望的不同。

**解决方案**：始终将 webhook 的 event.data.object 当作 `Record<string, any>` 处理，兼容新旧两种字段格式。不要依赖 SDK 的 TypeScript 类型来处理 webhook 数据。

### 坑-6：constructEvent vs constructEventAsync（Deno）

**问题**：Stripe Node.js SDK 的 `constructEvent()` 使用 Node 的 `crypto.createHmac()`，Deno 中不可用。异步版本 `constructEventAsync()` 也可能因运行时差异而失败。

**解决方案**：使用 WebCrypto API 手动验证 HMAC-SHA256（所有 Deno 运行时通用）：
```typescript
const key = await crypto.subtle.importKey(
  'raw', encoder.encode(secret),
  { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']
);
const sig = await crypto.subtle.sign('HMAC', key, encoder.encode(signedPayload));
```

**关键**：签名必须对 **原始 body 文本** 计算，不是解析后的 JSON。

### 坑-7：手动订阅不会自动过期

**问题**：手动订阅（微信/支付宝）没有 Stripe 侧的订阅对象，因此不会收到 `customer.subscription.deleted` webhook。订阅在数据库中永远保持 "ACTIVE"。

**解决方案**：pg_cron 定时任务自动过期 + 降级套餐：
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

### 坑-8：钱包拆分的连锁改动

**问题**：给钱包添加 `topup_balance` 字段后，**每一个**涉及钱包操作的 RPC 函数都需要同步更新。漏改任何一个函数都会导致 `balance_after` 计算错误。

**受影响函数**（必须同时更新）：
1. `billing_deduct_credits` — 双桶扣减逻辑
2. `billing_refund_credits` — 退款目标（始终退到 `balance`）
3. `billing_grant_subscription_credits` — 仅重置 `balance`，保留 `topup_balance`
4. `billing_activate_manual_subscription` — 仅设置 `balance`
5. `billing_renew_manual_subscription` — 仅设置 `balance`
6. `billing_admin_adjust_credits` — 操作 `topup_balance`
7. `billing_topup_credits` — 充值到 `topup_balance`

**规则**：流水中的 `balance_after` 必须始终等于 `balance + topup_balance`。

---

## 幂等模式速查

| 操作 | 幂等键 | 检查方式 |
|-----|-------|---------|
| Webhook 事件 | `stripe_event_id` | `stripe_webhook_events` 表唯一约束 |
| 积分发放（续费） | `period_start + tier` | `pay_subscription` 上的 `last_grant_period_start + last_grant_tier` |
| 积分发放（升级） | `period_start + 新tier` | 同上复合键 — 不同 tier = 新发放 |
| 退款 | `biz_id + USAGE_REFUND` | 检查 `billing_wallet_tx` 中是否已有 USAGE_REFUND |
| 加购充值 | `checkout_session_id + RECHARGE` | 检查 `billing_wallet_tx` 中是否已有 RECHARGE |
| 手动订阅续费 | `renew_ + checkout_session_id` | 检查 `billing_wallet_tx` 中的 biz_id |

---

## 调试速查表

| 症状 | 可能原因 | 修复方案 |
|-----|---------|---------|
| 快速 401（< 100ms） | 网关 JWT 拒绝（ES256/HS256 不匹配） | 用 `--no-verify-jwt` 部署 |
| 慢 401（> 500ms） | 函数内 `getUser()` 失败 | 检查 token 格式、auth 配置 |
| Webhook 400 | 签名不匹配 | 用原始 body 验证，不是解析后的 JSON |
| Webhook 200 但无效果 | 事件去重阻止了处理 | 检查 `stripe_webhook_events` 表 |
| 积分未发放 | 幂等检查阻止了 | 检查 `last_grant_period_start + last_grant_tier` |
| 重复订阅记录 | 用 INSERT 而非 UPSERT | 对 `stripe_subscription_id` 使用 UPSERT |
| 手动订阅永不过期 | 无自动过期机制 | 添加 pg_cron 定时任务 |
| `balance_after` 不对 | 钱包拆分未同步到所有 RPC | 同时更新全部 7 个钱包函数 |
| CORS 错误 | Origin 不在白名单 | 检查 `ALLOWED_ORIGIN` 环境变量，处理 www 变体 |
| 支付成功但没加积分 | Webhook 未配置或处理失败 | 检查 Stripe Dashboard 的 webhook 日志 |

---

## 测试与质量保证

### Stripe 测试模式配置

1. **API 密钥**：使用 [Stripe Dashboard > Developers > API Keys](https://dashboard.stripe.com/test/apikeys) 中的 `sk_test_*` / `pk_test_*`
2. **测试卡号**：
   - 成功：`4242 4242 4242 4242`（任意未来日期、任意 CVC）
   - 拒绝：`4000 0000 0000 0002`
   - 3D Secure：`4000 0025 0000 3155`
   - 余额不足：`4000 0000 0000 9995`

### Stripe CLI 本地 Webhook 测试

```bash
# 安装
brew install stripe/stripe-cli/stripe

# 登录（仅首次）
stripe login

# 转发 webhook 到本地 Edge Function（或 Next.js 端点）
stripe listen --forward-to http://localhost:54321/functions/v1/pay-stripe-webhook

# CLI 会打印 webhook 签名密钥（whsec_...）— 设为 STRIPE_WEBHOOK_SECRET

# 触发特定事件进行测试
stripe trigger checkout.session.completed
stripe trigger customer.subscription.updated
stripe trigger invoice.paid
stripe trigger customer.subscription.deleted
```

### 测试场景清单

**订阅流程**：
- [ ] 新用户注册 → 触发器自动发放 30 免费积分
- [ ] 免费用户 → Pro 订阅结账 → 积分到账
- [ ] Pro → Premium 升级（同周期）→ 累加积分，不重置
- [ ] Premium → Pro 降级尝试 → 被阻止（三层防护）
- [ ] 活跃订阅 → 取消 → 状态变为 `canceled`（当前周期结束前仍可用）
- [ ] 已取消订阅 → 恢复 → 状态回到 `active`
- [ ] 周期续费 → `invoice.paid` → 积分重置为新周期额度

**一次性加购流程**：
- [ ] 加购结账 → `checkout.session.completed` → `topup_balance` 增加
- [ ] 重复 webhook → 幂等机制阻止重复充值

**积分扣减**：
- [ ] 有足够积分生成 → 先扣 `balance`，再扣 `topup_balance`
- [ ] 总积分不足 → 返回错误，不进行部分扣减
- [ ] 并发扣减 → `FOR UPDATE` 行锁防止竞态条件

**手动订阅（如适用）**：
- [ ] 手动订阅激活 → 积分发放到 `balance`
- [ ] 手动订阅过期 → pg_cron 设置 `status = 'expired'`
- [ ] 手动订阅续费 → 新结账，发放新积分

**边界情况**：
- [ ] Webhook 在结账跳转完成前到达 → 用户刷新后看到积分
- [ ] Stripe API 版本不匹配 → `extractPeriod()` 兼容两种格式
- [ ] Edge Function 冷启动 → 限流仍然有效（基于数据库）

### Stripe Test Clocks（时间旅行测试）

用于不等待真实计费周期的订阅生命周期测试：

```bash
# 通过 Stripe Dashboard 创建 Test Clock：
# Stripe Dashboard > Developers > Test Clocks > New

# 创建关联到该 clock 的客户
# 为客户订阅一个计划
# 推进时钟来模拟：
#   - 试用期结束
#   - 首次续费
#   - 支付失败 + 重试周期
#   - 周期结束时取消订阅
```

Test Clocks 让你无需等待 30 天即可验证 `invoice.paid`、`customer.subscription.updated` 和过期事件的正确触发。

### 幂等回归测试

对每种 webhook 事件类型重放两次，验证：
1. 第一次调用：正常处理，返回 200
2. 第二次调用（相同 `event.id`）：被 `stripe_webhook_events` 去重跳过，返回 200
3. 第二次调用后数据库状态不变

```bash
# 使用 Stripe CLI 重放事件：
stripe events resend evt_xxxxxxxxxxxxx
```

---

## 部署检查清单

### Supabase
- [ ] 所有迁移文件按顺序执行
- [ ] 所有表启用 RLS
- [ ] `billing_handle_new_user` 触发器在 `auth.users` 上就位
- [ ] 启用 pg_cron 扩展（用于清理 + 手动订阅过期）
- [ ] 所有 Edge Function 用 `--no-verify-jwt` 部署

### Stripe
- [ ] Webhook 端点配置了正确的 URL
- [ ] 订阅事件: `checkout.session.completed`, `customer.subscription.created`, `customer.subscription.updated`, `invoice.paid`, `customer.subscription.deleted`
- [ ] Webhook 签名密钥设为 `STRIPE_WEBHOOK_SECRET` 环境变量
- [ ] 记录账号默认 API 版本（可能与 SDK 版本不同）

### Edge Function 环境变量
- [ ] `SUPABASE_URL`
- [ ] `SUPABASE_SERVICE_ROLE_KEY`
- [ ] `STRIPE_SECRET_KEY`
- [ ] `STRIPE_WEBHOOK_SECRET`
- [ ] `APP_BASE_URL`（如 `https://your-app.com`）
- [ ] `ALLOWED_ORIGIN`（CORS 源，开发环境可用 `*`）

### 前端
- [ ] `NEXT_PUBLIC_SUPABASE_URL` 已设置
- [ ] `NEXT_PUBLIC_SUPABASE_ANON_KEY` 已设置
- [ ] 定价常量与 Stripe Dashboard 中的 price ID 一致
- [ ] 回调 URL 使用相对路径（服务端拼接）

---

## 关键设计决策

1. **积分计费** 而非按次计价：UX 更简单，成本可预测
2. **双桶钱包** 而非单一余额：订阅积分每月重置，加购积分永久保留
3. **数据库 RPC** 而非应用层逻辑：原子事务，无竞态条件
4. **`FOR UPDATE` 行锁**：防止并发扣减/发放冲突
5. **Webhook 驱动状态同步**：Stripe 是自动续费订阅的唯一真相源
6. **数据库限流**：抗 Deno 冷启动 + 多实例部署共享
7. **三层降级防护**：纵深防御（前端、Edge Function、数据库）
8. **手动 HMAC 验证**：在所有 Deno 运行时通用（不依赖 Node.js crypto）

---

## 监控与可观测性

### Edge Function 日志

Supabase Edge Function 的日志在 Supabase Dashboard（Logs > Edge Functions）中查看。使用结构化 JSON 日志便于检索：

```typescript
// 在 Edge Function 处理器中 — 使用结构化 JSON 日志
console.log(JSON.stringify({
  event: 'webhook_processed',
  stripe_event_id: event.id,
  type: event.type,
  user_id: userId,
  credits_granted: amount,
  duration_ms: Date.now() - startTime,
}))

// 错误日志带上下文
console.error(JSON.stringify({
  event: 'webhook_error',
  stripe_event_id: event.id,
  error: error.message,
  stack: error.stack,
}))
```

### Stripe Webhook 健康监控

在 Stripe Dashboard 中监控 webhook 投递健康度：

1. **Stripe Dashboard > Developers > Webhooks > [你的端点]**
   - 检查投递成功率（目标：> 99%）
   - 查看失败事件及其错误响应
   - 修复处理器后重发失败事件

2. **关键 Webhook 指标**：
   - 待处理事件队列（应接近 0）
   - 平均响应时间（应 < 5s，最好 < 1s）
   - 失败投递数量（稳态应为 0）

3. **Stripe 自动重试机制**：失败事件会在 3 天内以指数退避重试。如果端点临时宕机，事件最终会到达。

### 关键计费指标（SQL 查询）

```sql
-- 各层级活跃订阅数
SELECT tier, status, count(*)
FROM pay_subscription
GROUP BY tier, status;

-- 每日消耗（从钱包交易记录）
SELECT date_trunc('day', created_at) AS day,
       sum(amount) AS total_credits_deducted
FROM billing_wallet_tx
WHERE tx_type = 'USAGE'
GROUP BY day ORDER BY day DESC LIMIT 30;

-- 低余额用户（潜在流失信号）
SELECT w.user_id, w.balance, w.topup_balance,
       p.display_name, p.tier
FROM billing_wallet w
JOIN member_profile p ON p.id = w.user_id
WHERE w.balance + w.topup_balance < 10
  AND p.tier != 'free'
ORDER BY w.balance + w.topup_balance;

-- Webhook 处理延迟
SELECT event_type,
       avg(extract(epoch from processed_at - created_at)) AS avg_lag_seconds
FROM stripe_webhook_events
WHERE created_at > now() - interval '7 days'
GROUP BY event_type;

-- 失败的 webhook 事件（未处理）
SELECT * FROM stripe_webhook_events
WHERE processed_at IS NULL
  AND created_at > now() - interval '3 days'
ORDER BY created_at DESC;
```

### 告警策略

| 告警 | 条件 | 严重级别 | 处理方式 |
|-----|------|---------|---------|
| Webhook 队列积压 | 待处理事件 > 10 | 高 | 检查 Edge Function 日志，必要时重新部署 |
| 积分余额不一致 | `balance_after != balance + topup_balance` | 严重 | 运行对账查询，检查 RPC 逻辑 |
| 订阅无钱包 | `pay_subscription` 存在但无 `billing_wallet` | 严重 | 检查 `billing_handle_new_user` 触发器 |
| Webhook 高延迟 | 平均响应 > 5s | 中 | 检查慢 SQL 查询、冷启动问题 |
| 重复订阅记录 | 单用户多条活跃订阅 | 高 | 检查 webhook 处理器中的 UPSERT 逻辑 |

### 生产环境健康检查查询

定期运行以验证系统完整性：

```sql
-- 完整性检查：每个订阅用户都有钱包
SELECT s.user_id, s.tier, s.status
FROM pay_subscription s
LEFT JOIN billing_wallet w ON w.user_id = s.user_id
WHERE w.user_id IS NULL AND s.status = 'active';
-- 应返回 0 行

-- 完整性检查：balance_after 一致性
SELECT t.id, t.balance_after,
       w.balance + w.topup_balance AS actual_total
FROM billing_wallet_tx t
JOIN billing_wallet w ON w.user_id = t.user_id
WHERE t.id = (
  SELECT max(id) FROM billing_wallet_tx t2
  WHERE t2.user_id = t.user_id
)
AND t.balance_after != w.balance + w.topup_balance;
-- 应返回 0 行
```

---

## 相关 Skills

| Skill | 何时使用 |
|-------|---------|
| **mvp-billing-system** | 本 skill 的英文版（内容相同） |
| **stripe-payments** | Stripe 支付集成入门版（Next.js Route Handler 模式，无积分/钱包） |
| **supabase-developer** | Supabase 全功能参考（Auth、Database、Storage、Real-time） |
| **supabase-gemini-deploy** | Edge Function 部署排查手册（JWT、CORS、401/500 诊断） |
| **deploy-gate** | 部署前六关门禁检查（编译、环境变量、Git 等） |
| **nextjs-fullstack** | Next.js App Router 开发模式（Server Components、Actions、中间件） |
| **saas-quickstart** | 从 starter kit 快速启动（5步上线） |
| **project-scaffold** | 完整架构文档生成 + 约束代码骨架 |
