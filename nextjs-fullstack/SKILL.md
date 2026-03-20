---
name: nextjs-fullstack
description: Next.js App Router 全栈开发skill。覆盖App Router、Server Components、Server Actions、Route Handlers、缓存策略、中间件。当提到"Next.js"、"页面"、"路由"、"Server Action"、"全栈"时触发。
---

# Next.js App Router 全栈开发

> 适用于 Next.js 14/15+（App Router）。如使用 Pages Router 请参考官方迁移指南。

## 第一原则
- **永远先读 node_modules/next/dist/docs/** — 训练数据过时，bundled docs是真相来源
- **Server Components是默认** — 只在需要交互时用'use client'
- **Server Actions替代API Route** — 表单/mutation优先用Server Actions
- **Route Handlers用于Webhook** — 外部服务回调用Route Handlers

## 项目结构
```
src/
├── app/                    # App Router
│   ├── (auth)/             # 认证相关页面组
│   │   ├── login/page.tsx
│   │   └── signup/page.tsx
│   ├── (dashboard)/        # 登录后页面组
│   │   ├── layout.tsx      # Dashboard布局（含侧边栏）
│   │   └── page.tsx
│   ├── api/
│   │   └── webhooks/       # Webhook Route Handlers
│   ├── layout.tsx          # 根布局
│   └── page.tsx            # Landing page
├── components/             # 共享UI组件
│   └── ui/                 # shadcn/ui组件
├── features/               # 按功能组织的业务代码
│   ├── auth/
│   ├── billing/
│   └── [your-tool]/        # 你的工具功能
├── lib/                    # 工具库
│   ├── supabase/
│   └── stripe/
└── types/                  # TypeScript类型
```

## Server Actions模式
```typescript
// features/items/actions.ts
'use server'
import { createClient } from '@/lib/supabase/server'
import { revalidatePath } from 'next/cache'
import { z } from 'zod'

const CreateItemSchema = z.object({
  name: z.string().min(1).max(100),
  description: z.string().optional(),
})

export async function createItem(formData: FormData) {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) throw new Error('Unauthorized')

  const parsed = CreateItemSchema.safeParse({
    name: formData.get('name'),
    description: formData.get('description'),
  })
  if (!parsed.success) return { error: parsed.error.flatten() }

  const { error } = await supabase.from('items').insert({
    ...parsed.data,
    user_id: user.id,
  })
  if (error) return { error: error.message }

  revalidatePath('/dashboard')
  return { success: true }
}
```

## 认证保护（Middleware）
```typescript
// middleware.ts
import { createServerClient } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'

export async function middleware(request: NextRequest) {
  const response = NextResponse.next({ request })
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    { cookies: { /* cookie handlers */ } }
  )
  const { data: { user } } = await supabase.auth.getUser()

  if (!user && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  return response
}

export const config = { matcher: ['/dashboard/:path*'] }
```

## 数据获取模式
```typescript
// Server Component直接查询（推荐）
export default async function ItemsList() {
  const supabase = await createClient()
  const { data: items } = await supabase
    .from('items')
    .select('*')
    .order('created_at', { ascending: false })
  
  return <ul>{items?.map(item => <li key={item.id}>{item.name}</li>)}</ul>
}
```

## 性能指标目标
- TTFB < 200ms
- FCP < 1s
- LCP < 2.5s
- CLS < 0.1

## 部署到Vercel
```bash
npx vercel link
npx vercel env add NEXT_PUBLIC_SUPABASE_URL
npx vercel env add NEXT_PUBLIC_SUPABASE_ANON_KEY
npx vercel env add SUPABASE_SERVICE_ROLE_KEY
npx vercel env add STRIPE_SECRET_KEY
npx vercel env add STRIPE_WEBHOOK_SECRET
npx vercel --prod
```

---

## Related Skills

| Skill | 何时使用 |
|-------|---------|
| **supabase-developer** | Supabase 全功能参考（Auth、Database、Storage、Real-time） |
| **stripe-payments** | 快速集成 Stripe 订阅门控（Next.js Route Handler 模式） |
| **mvp-billing-system** | 生产级计费系统（Edge Function + 积分钱包 + 双轨支付） |
| **deploy-gate** | 部署前六关门禁检查 |
