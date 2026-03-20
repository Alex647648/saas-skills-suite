---
name: saas-quickstart
description: SaaS产品快速启动skill。从starter kit到上线的完整流程。当提到"新建SaaS"、"创建项目"、"从零开始"、"快速上线"、"SaaS模板"时触发。
---

# SaaS 快速启动

## 推荐Starter Kit
**KolbySisk/next-supabase-stripe-starter**（开源免费，MIT协议）
预建：用户注册/登录 + OAuth + Stripe订阅 + Webhook同步 + RLS + Dashboard骨架

## 5步上线

### Step 1: 初始化（10分钟）
```bash
git clone https://github.com/KolbySisk/next-supabase-stripe-starter.git my-app
cd my-app && npm install
cp .env.local.example .env.local
```

### Step 2: 配置服务（30分钟）
1. **Supabase**: app.supabase.com → New Project → 拿URL和Keys
2. **Stripe**: dashboard.stripe.com → Developers → API Keys
3. 填入 .env.local
4. 运行migration: `npx supabase db push`
5. 配置Stripe产品: `stripe fixtures ./stripe-fixtures.json --api-key sk_test_xxx`

### Step 3: 加你的工具功能（1-3天）
在 `src/features/[your-tool]/` 下创建：
- actions.ts — Server Actions（业务逻辑）
- queries.ts — 数据查询
- components/ — UI组件
- types.ts — TypeScript类型

### Step 4: 支付门控
```typescript
import { getActiveSubscription } from '@/features/billing/queries'

export default async function ToolPage() {
  const sub = await getActiveSubscription()
  if (!sub) return <PricingPage />
  return <YourTool />
}
```

### Step 5: 部署
```bash
npx vercel --prod
# Stripe Dashboard → Webhooks → Add endpoint → https://your-app.vercel.app/api/webhooks
# Vercel Dashboard → Domains → 添加自定义域名
```

## 需要的账号
- [x] GitHub（代码托管）
- [ ] Supabase（免费tier够用）
- [ ] Stripe（测试模式免费）
- [ ] Vercel（免费tier够用）
- [ ] 域名（可选，Vercel提供.vercel.app子域名）

---

## Related Skills

| Skill | 何时使用 |
|-------|---------|
| **project-scaffold** | 需要完整的架构文档和约束代码骨架时（比本 skill 更重量级） |
| **mvp-billing-system** | 从零构建积分计费 + 双轨支付系统 |
| **stripe-payments** | 快速集成 Stripe 订阅门控（入门版） |
| **nextjs-fullstack** | Next.js App Router 全栈开发模式 |
| **supabase-developer** | Supabase 全功能参考（Auth、RLS、Storage、Real-time） |
| **deploy-gate** | 上线前的六关门禁检查 |
