---
name: stripe-payments
description: Stripe支付集成入门skill，专注于Next.js Route Handler + Supabase场景。覆盖Checkout Session、订阅管理、Webhook同步、支付门控。适合简单订阅门控的快速实现。当提到"支付入门"、"Stripe快速集成"时触发。
---

# Stripe + Supabase 支付集成（入门版）

> **定位说明**：本 skill 是**入门级模板**，适合简单的订阅门控场景（一种支付方式、无积分系统）。
> 架构为 Next.js Route Handler + Server Action，适合小型项目快速上线。
>
> 如需以下能力，请使用 **mvp-billing-system**（或中文版 **mvp-billing-system-cn**）：
> - 积分/钱包计费系统
> - 双轨支付（银行卡 + 微信/支付宝）
> - Supabase Edge Function 架构
> - 幂等保障、webhook 去重、金额交叉验证
> - 手动订阅生命周期管理
> - 生产级安全加固（数据库限流、审计日志）

## 核心原则
1. **永不前端处理敏感支付数据** — 使用Stripe Checkout Session
2. **Stripe是唯一的支付真相来源** — 数据库通过Webhook同步
3. **Webhook handler必须幂等** — 用event ID去重
4. **test mode开发，live mode上线** — 永远分开配置

## 支付架构
```
用户点击订阅 → Server Action创建Checkout Session →
跳转Stripe Checkout → 用户付款 → Stripe发Webhook →
Route Handler接收 → 验签 → 同步到Supabase → 用户获得权限
```

## 实现清单

### 1. 产品配置（stripe-fixtures.json）
```json
{
  "_meta": { "template_version": "1" },
  "fixtures": [
    {
      "name": "basic_product",
      "path": "/v1/products",
      "method": "post",
      "params": {
        "name": "Basic Plan",
        "metadata": { "features": "core", "max_projects": "3" }
      }
    },
    {
      "name": "basic_price",
      "path": "/v1/prices",
      "method": "post",
      "params": {
        "product": "${basic_product:id}",
        "unit_amount": 990,
        "currency": "usd",
        "recurring": { "interval": "month" }
      }
    }
  ]
}
```
运行: `stripe fixtures ./stripe-fixtures.json --api-key sk_test_xxx`

### 2. Checkout Session（Server Action）
```typescript
'use server'
import { stripe } from '@/lib/stripe'
import { createClient } from '@/lib/supabase/server'

export async function createCheckoutSession(priceId: string) {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) throw new Error('Not authenticated')

  const session = await stripe.checkout.sessions.create({
    customer_email: user.email,
    line_items: [{ price: priceId, quantity: 1 }],
    mode: 'subscription',
    success_url: `${process.env.NEXT_PUBLIC_SITE_URL}/dashboard?success=true`,
    cancel_url: `${process.env.NEXT_PUBLIC_SITE_URL}/pricing?canceled=true`,
    subscription_data: { metadata: { supabase_user_id: user.id } },
  })
  return { url: session.url }
}
```

### 3. Webhook Handler（Route Handler）
```typescript
// app/api/webhooks/stripe/route.ts
import { stripe } from '@/lib/stripe'
import { createAdminClient } from '@/lib/supabase/admin'
import { headers } from 'next/headers'

export async function POST(req: Request) {
  const body = await req.text()  // raw body for verification
  const sig = (await headers()).get('stripe-signature')!
  
  let event
  try {
    event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!)
  } catch (err) {
    return new Response('Webhook signature verification failed', { status: 400 })
  }

  const supabase = createAdminClient()

  switch (event.type) {
    case 'checkout.session.completed': {
      const session = event.data.object
      const userId = session.subscription_data?.metadata?.supabase_user_id
      await supabase.from('subscriptions').upsert({
        user_id: userId,
        stripe_subscription_id: session.subscription,
        status: 'active',
        price_id: session.line_items?.data[0]?.price?.id,
      })
      break
    }
    case 'customer.subscription.updated':
    case 'customer.subscription.deleted': {
      const sub = event.data.object
      await supabase.from('subscriptions').update({
        status: sub.status,
        cancel_at: sub.cancel_at ? new Date(sub.cancel_at * 1000).toISOString() : null,
      }).eq('stripe_subscription_id', sub.id)
      break
    }
    case 'invoice.payment_failed': {
      const invoice = event.data.object
      await supabase.from('subscriptions').update({
        status: 'past_due',
      }).eq('stripe_subscription_id', invoice.subscription)
      break
    }
  }
  return new Response('OK', { status: 200 })
}
```

### 4. 订阅状态检查
```typescript
// lib/subscription.ts
import { createClient } from '@/lib/supabase/server'

export async function getActiveSubscription() {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return null

  const { data } = await supabase
    .from('subscriptions')
    .select('*')
    .eq('user_id', user.id)
    .eq('status', 'active')
    .single()
  return data
}

// 在页面中使用
export default async function PremiumFeature() {
  const sub = await getActiveSubscription()
  if (!sub) return <UpgradePrompt />
  return <YourPremiumTool />
}
```

### 5. 客户自助门户
```typescript
'use server'
export async function createPortalSession() {
  const session = await stripe.billingPortal.sessions.create({
    customer: customerId,
    return_url: `${process.env.NEXT_PUBLIC_SITE_URL}/dashboard`,
  })
  return { url: session.url }
}
```

### 6. Supabase表结构
```sql
CREATE TABLE subscriptions (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID REFERENCES auth.users NOT NULL,
  stripe_subscription_id TEXT UNIQUE,
  stripe_customer_id TEXT,
  status TEXT NOT NULL DEFAULT 'inactive',
  price_id TEXT,
  cancel_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE subscriptions ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users read own subscription"
  ON subscriptions FOR SELECT
  USING (auth.uid() = user_id);
```

## 环境变量清单
```
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_xxx
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
```

## 安全检查
- [ ] Webhook使用raw body验签（不是parsed JSON）
- [ ] STRIPE_SECRET_KEY仅服务端使用
- [ ] subscription表有RLS
- [ ] Webhook handler对每个event type都是幂等的
- [ ] 测试使用stripe test clock验证订阅生命周期

---

## Related Skills

| Skill | 何时使用 |
|-------|---------|
| **mvp-billing-system** | 需要积分计费、双轨支付、Edge Function 架构时（生产级） |
| **mvp-billing-system-cn** | 同上，中文版 |
| **supabase-developer** | Supabase 全功能参考（Auth、RLS、Storage、Real-time） |
| **saas-quickstart** | 从零用 starter kit 快速启动 SaaS |
| **nextjs-fullstack** | Next.js App Router 全栈开发模式 |
