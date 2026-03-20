---
name: Supabase Gemini Deploy
description: Supabase Edge Function 部署 Gemini AI 代理的完整排查与修复 skill。覆盖 JWT 认证（ES256/HS256）、pgcrypto 扩展缺失、CORS、API Key 配置、计费 RPC 错误、速率限制等所有常见问题。当遇到 Edge Function 401/403/500 错误、Gemini 调用失败、计费系统错误时触发。
version: 1.0.0
category: deployment
tags:
  - supabase
  - gemini
  - edge-functions
  - deployment
  - debugging
  - jwt
  - authentication
triggers:
  - supabase-gemini-deploy
  - gemini deploy
  - edge function 401
  - edge function 500
  - gemini proxy
  - 计费系统错误
  - billing error
  - jwt es256
---

# Supabase Edge Function + Gemini AI 部署排查 Skill

## 适用场景

在 Supabase Edge Functions 中部署 Gemini AI 代理时遇到的所有问题。包括但不限于：
- Edge Function 返回 401/403/500
- Gemini API 调用失败
- 计费系统（billing RPC）报错
- CORS 跨域问题
- JWT 认证不通过
- 数据库函数缺失或权限问题

---

## 核心知识：三层认证架构

Supabase Edge Function 调用链有 **三层认证**，每层都可能独立返回 401：

```
客户端 (gemini-service.ts)
  │
  │  Authorization: Bearer <user_access_token>
  │  apikey: <anon_key>
  │
  ▼
[第1层] Supabase API Gateway (Kong)
  │  - 验证 apikey header 识别项目
  │  - 默认验证 Authorization JWT 签名
  │  - !! 只支持 HS256，不支持 ES256 !!
  │
  ▼
[第2层] Edge Function 代码 (Deno)
  │  - supabase.auth.getUser(token)
  │  - 支持 ES256 和 HS256
  │
  ▼
[第3层] 数据库 RPC (PostgreSQL)
  │  - billing_deduct_credits()
  │  - search_path 和扩展可用性
  │
  ▼
[第4层] 外部 API (Gemini)
  - GEMINI_API_KEY 验证
  - 速率限制 (15 RPM free tier)
```

---

## 问题速查表

### 按 HTTP 状态码分类

| 状态码 | 响应时间 | 错误信息 | 根因 | 修复方案 |
|--------|---------|---------|------|---------|
| 401 | < 100ms | 无 JSON body 或 Gateway 格式 | Gateway JWT 验证失败（ES256/HS256 不匹配） | `--no-verify-jwt` 部署 |
| 401 | > 500ms | `{"error_code":"UNAUTHORIZED"}` | 函数级 `getUser()` 失败 | 检查用户是否已登录、token 是否过期 |
| 403 | 任意 | `{"error_code":"MODEL_NOT_AVAILABLE"}` | 用户 tier 无权使用该模型 | 检查 MODEL_ACCESS 配置和 member_profile.tier |
| 404 | < 50ms | `Not Found` | Edge Function 未部署 | `supabase functions deploy <name>` |
| 429 | 任意 | `{"error_code":"RATE_LIMITED"}` | 60秒内超过30次请求 | 等待后重试 |
| 500 | 任意 | `{"error_code":"BILLING_ERROR"}` | RPC 函数执行失败 | 见下方「计费系统错误」 |
| 500 | 任意 | `{"error_code":"GEMINI_ERROR"}` | Gemini API 调用失败 | 检查 GEMINI_API_KEY、API 配额 |
| 500 | 任意 | `"服务器配置错误"` | GEMINI_API_KEY 未设置 | `supabase secrets set GEMINI_API_KEY=xxx` |
| CORS | N/A | `Access-Control-Allow-Origin` 错误 | ALLOWED_ORIGIN 不匹配 | `supabase secrets set ALLOWED_ORIGIN=*` |

---

## 问题 1：401 — ES256 vs HS256 JWT 不匹配（最常见）

### 症状

- 所有 POST 请求返回 401
- OPTIONS 预检返回 200（CORS 正常）
- 快速 401（< 100ms）说明是 Gateway 层拦截

### 诊断

浏览器 Console 检查 DEBUG 日志：

```
[DEBUG] access_token (first 50): eyJhbGciOiJFUzI1NiIs...  ← ES256
[DEBUG] anon_key (first 50):     eyJhbGciOiJIUzI1NiIs...  ← HS256
```

**关键判断**：如果 access_token 的 `alg` 是 `ES256`，而 anon_key 的 `alg` 是 `HS256`，就是此问题。

### 原因

Supabase 新版 Auth（2025+）使用 **ES256**（ECDSA 非对称签名）签发用户 access_token，但 Edge Function API Gateway 默认用 **HS256** 验证 JWT。算法不匹配，Gateway 在函数代码运行前就返回 401。

### 修复

所有 Edge Functions 加 `--no-verify-jwt` 部署，让 Gateway 跳过 JWT 验证，由函数内部 `getUser(token)` 处理认证（它支持 ES256）：

```bash
supabase functions deploy gemini-proxy --no-verify-jwt
supabase functions deploy pay-create-checkout --no-verify-jwt
supabase functions deploy pay-stripe-webhook --no-verify-jwt
supabase functions deploy pay-manage-subscription --no-verify-jwt
supabase functions deploy billing-admin --no-verify-jwt
```

### 安全性说明

`--no-verify-jwt` 只是跳过 Gateway 层的 JWT 签名校验。函数内部仍然通过 `supabase.auth.getUser(token)` 验证用户身份（调用 Supabase Auth API，能正确处理 ES256）。安全性不受影响。

### 如何区分 Gateway 401 和 Function 401

| 特征 | Gateway 401 | Function 401 |
|------|-------------|--------------|
| 响应时间 | < 100ms | > 500ms |
| Response body | 无 JSON 或 Kong 格式 | `{"ok":false,"error_code":"UNAUTHORIZED"}` |
| 原因 | JWT 签名算法不匹配 | token 过期或无效 |

---

## 问题 2：500 BILLING_ERROR — pgcrypto / search_path 问题

### 症状

```json
{"ok":false,"error_code":"BILLING_ERROR","message":"计费系统错误"}
```

### 诊断

直接调用 RPC 测试：

```bash
curl -s "https://<project-ref>.supabase.co/rest/v1/rpc/billing_deduct_credits" \
  -H "apikey: <service_role_key>" \
  -H "Authorization: Bearer <service_role_key>" \
  -H "Content-Type: application/json" \
  -d '{"p_user_id":"<user-uuid>","p_amount":0.5,"p_action":"chat","p_model":"gemini-3-flash-preview","p_biz_id":"test-001"}'
```

常见错误响应：

| 错误 | 根因 | 修复 |
|------|------|------|
| `function gen_random_bytes(integer) does not exist` | pgcrypto 扩展不在 search_path | 见下方修复 |
| `relation "member_profile" does not exist` | 迁移未推送 | `supabase db push` |
| `column "tier" check constraint violated` | tier='admin' 但 CHECK 不含 admin | 更新 CHECK 约束 |

### 修复：gen_random_bytes 不存在

**原因**：`billing_generate_tx_no()` 使用 `gen_random_bytes(8)`（来自 pgcrypto 扩展），但 RPC 函数的 `SET search_path = public` 不包含 `extensions` schema（Supabase 的 pgcrypto 安装在 extensions schema）。

**方案 A**（推荐）：替换为内置函数，不依赖 pgcrypto：

```sql
CREATE OR REPLACE FUNCTION billing_generate_tx_no()
RETURNS TEXT
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN 'WT-' || to_char(now(), 'YYYYMMDDHH24MISS') || '-'
    || upper(substr(replace(gen_random_uuid()::text, '-', ''), 1, 8));
END;
$$;
```

`gen_random_uuid()` 是 PostgreSQL 13+ 内置函数，不需要任何扩展。

**方案 B**：扩展 search_path：

```sql
-- 对所有 SECURITY DEFINER 函数添加 extensions 到 search_path
CREATE OR REPLACE FUNCTION billing_deduct_credits(...)
...
SET search_path = public, extensions  -- 加上 extensions
AS $$ ... $$;
```

### 预防措施

编写 Supabase SECURITY DEFINER 函数时，始终使用 `SET search_path = public, extensions`，以确保能访问 pgcrypto、uuid-ossp 等扩展。

---

## 问题 3：500 — GEMINI_API_KEY 未配置

### 症状

```json
{"ok":false,"error_code":"INTERNAL_ERROR","message":"服务器配置错误"}
```

或 Edge Function 日志显示 `GEMINI_API_KEY is not set`。

### 修复

```bash
# 设置 Gemini API Key（从 https://aistudio.google.com/apikeys 获取）
supabase secrets set GEMINI_API_KEY=AIza你的完整Key

# 验证
supabase secrets list
```

### 相关环境变量清单

| 变量名 | 必须 | 说明 | 设置方式 |
|--------|------|------|---------|
| `GEMINI_API_KEY` | 是 | Google AI Studio API Key | `supabase secrets set` |
| `ALLOWED_ORIGIN` | 是 | CORS 允许的前端域名 | `supabase secrets set`（开发时设 `*`） |
| `APP_BASE_URL` | 是 | 应用 URL（Stripe 回调用） | `supabase secrets set` |
| `STRIPE_SECRET_KEY` | 支付时 | Stripe Secret Key | `supabase secrets set` |
| `STRIPE_WEBHOOK_SECRET` | 支付时 | Stripe Webhook 签名密钥 | `supabase secrets set` |
| `SUPABASE_URL` | 自动 | Supabase 自动注入 | 无需手动设置 |
| `SUPABASE_SERVICE_ROLE_KEY` | 自动 | Supabase 自动注入 | 无需手动设置 |

---

## 问题 4：401 — 用户未登录 / Session 丢失

### 症状

- 401 但响应较慢（> 500ms）
- Response body 包含 `{"error_code":"UNAUTHORIZED","message":"Invalid token"}`
- 浏览器 DEBUG 日志 `session exists: false`

### 诊断

检查客户端 DEBUG 日志：

```javascript
// gemini-service.ts 中的 DEBUG 输出
[DEBUG] session exists: false          ← 未登录
[DEBUG] access_token (first 50): eyJ... ← 检查是 anon_key 还是 user token
[DEBUG] sb-* cookies: []               ← 空 = 无 auth cookie
```

### 常见原因

1. **用户未登录**：客户端 fallback 到 anon_key，`getUser(anon_key)` 必然失败
2. **Session 过期**：access_token 超过 `jwt_expiry`（默认 3600s），refresh 失败
3. **Cookie 域不匹配**：部署后域名变了，auth cookie 设在旧域名上
4. **Supabase Auth URL Configuration 未更新**：Site URL 和 Redirect URLs 没改为生产域名

### 修复

```
1. 确保用户已登录（检查 /login 页面是否正常工作）
2. Supabase Dashboard → Authentication → URL Configuration
   - Site URL: https://你的生产域名
   - Redirect URLs: 添加 https://你的生产域名/**
3. 清除浏览器 cookies 后重新登录
```

---

## 问题 5：404 — member_profile / billing_wallet 缺失

### 症状

```json
{"ok":false,"error_code":"USER_NOT_FOUND","message":"Profile not found"}
```

### 原因

用户注册时 `handle_new_user()` trigger 只创建 `users` 表记录，没有自动创建 `member_profile` 和 `billing_wallet`。

### 诊断

```bash
# 检查用户是否有 profile
curl -s "https://<ref>.supabase.co/rest/v1/member_profile?user_id=eq.<uid>" \
  -H "apikey: <service_role_key>" \
  -H "Authorization: Bearer <service_role_key>"

# 检查用户是否有 wallet
curl -s "https://<ref>.supabase.co/rest/v1/billing_wallet?user_id=eq.<uid>" \
  -H "apikey: <service_role_key>" \
  -H "Authorization: Bearer <service_role_key>"
```

### 修复

**临时**：手动插入（在 SQL Editor 中执行）：

```sql
INSERT INTO member_profile (user_id, tier, free_quota)
VALUES ('<user-uuid>', 'free', 3)
ON CONFLICT (user_id) DO NOTHING;

INSERT INTO billing_wallet (user_id, balance)
VALUES ('<user-uuid>', 0)
ON CONFLICT (user_id) DO NOTHING;
```

**永久**：更新 `handle_new_user()` trigger 自动创建 profile 和 wallet：

```sql
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS trigger AS $$
BEGIN
  INSERT INTO public.users (id, full_name, avatar_url)
  VALUES (new.id, new.raw_user_meta_data->>'full_name', new.raw_user_meta_data->>'avatar_url');

  INSERT INTO public.member_profile (user_id, tier, free_quota)
  VALUES (new.id, 'free', 3);

  INSERT INTO public.billing_wallet (user_id, balance)
  VALUES (new.id, 0);

  RETURN new;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

## 问题 6：Gemini API 限流 / 配额耗尽

### 症状

```json
{"ok":false,"error_code":"GEMINI_ERROR","message":"AI 服务暂时繁忙，请稍后再试"}
```

Edge Function 日志：`429 Resource has been exhausted` 或 `QUOTA_EXCEEDED`。

### 免费配额限制

| 限制项 | 免费额度 |
|--------|---------|
| 每分钟请求数 (RPM) | 15 |
| 每天请求数 (RPD) | 1500 |
| 每分钟 token 数 | 100万 |

### 修复

1. **等待**：免费配额每分钟/每天自动重置
2. **升级**：在 Google Cloud Console 启用付费，配额大幅提升
3. **重试逻辑**：Edge Function 已内置 `withRetry()` 指数退避，最多重试 3 次（仅针对 429）

---

## 问题 7：CORS 跨域错误

### 症状

浏览器 Console：
```
Access to fetch at 'https://xxx.supabase.co/functions/v1/gemini-proxy'
from origin 'https://your-app.vercel.app' has been blocked by CORS policy
```

### 修复

```bash
# 开发环境：允许所有域名
supabase secrets set ALLOWED_ORIGIN=*

# 生产环境：指定具体域名
supabase secrets set ALLOWED_ORIGIN=https://your-app.vercel.app
```

Edge Function 的 CORS 处理逻辑位于 `corsHeaders()` 函数中，检查 `origin` header 是否匹配 `ALLOWED_ORIGIN`。

---

## 问题 8：gen_log billing_type CHECK 约束冲突

### 症状

Admin 用户调用 Gemini 时报 500 错误，日志显示 CHECK constraint violation。

### 原因

`gen_log` 表的 `billing_type` CHECK 约束只允许 `'FREE_TRIAL'` 和 `'CREDITS'`，但 Edge Function 对 admin 用户写入 `'ADMIN'`。

### 修复

```sql
ALTER TABLE gen_log DROP CONSTRAINT IF EXISTS gen_log_billing_type_check;
ALTER TABLE gen_log ADD CONSTRAINT gen_log_billing_type_check
  CHECK (billing_type IN ('FREE_TRIAL', 'CREDITS', 'ADMIN'));
```

---

## 问题 9：member_profile tier CHECK 约束不含 admin

### 症状

执行 `UPDATE member_profile SET tier = 'admin'` 失败。

### 原因

原始 CHECK 约束：`CHECK (tier IN ('free','basic','pro','max'))`，不包含 `'admin'`。

### 修复

```sql
ALTER TABLE member_profile DROP CONSTRAINT IF EXISTS member_profile_tier_check;
ALTER TABLE member_profile ADD CONSTRAINT member_profile_tier_check
  CHECK (tier IN ('free', 'basic', 'pro', 'max', 'admin'));
```

---

## 完整部署检查清单

按顺序逐项执行：

```
[ ] 1. supabase link --project-ref <ref>
[ ] 2. supabase db push（或 SQL Editor 手动执行迁移）
[ ] 3. 验证表存在：member_profile, billing_wallet, gen_log, pay_subscription
[ ] 4. 验证 RPC 存在：billing_deduct_credits, billing_refund_credits
[ ] 5. 修复 billing_generate_tx_no（gen_random_uuid 替代 gen_random_bytes）
[ ] 6. 修复 CHECK 约束（member_profile.tier 含 admin、gen_log.billing_type 含 ADMIN）
[ ] 7. 更新 handle_new_user trigger（自动创建 profile + wallet）
[ ] 8. supabase secrets set GEMINI_API_KEY=AIza...
[ ] 9. supabase secrets set ALLOWED_ORIGIN=*
[ ] 10. supabase secrets set APP_BASE_URL=http://localhost:3000
[ ] 11. supabase functions deploy gemini-proxy --no-verify-jwt
[ ] 12. supabase functions deploy 其余 4 个 --no-verify-jwt
[ ] 13. 注册用户 → 检查 member_profile 和 billing_wallet 是否自动创建
[ ] 14. 测试 AI 聊天 → 检查 gen_log 是否有记录
[ ] 15. 测试 3 次免费额度 → 第 4 次应弹出付费弹窗
[ ] 16. 提升 admin → UPDATE member_profile SET tier='admin'
[ ] 17. admin 测试 → 应无限使用，不扣积分
```

---

## 调试工具箱

### 客户端 DEBUG 日志（已内置）

`gemini-service.ts` 中的 `callProxy()` 函数自带 DEBUG 输出：

```
[DEBUG] session exists: true/false
[DEBUG] access_token (first 50): eyJ...   ← 检查 alg 是 ES256 还是 HS256
[DEBUG] anon_key (first 50): eyJ...
[DEBUG] FUNCTION_URL: https://xxx.supabase.co/functions/v1
[DEBUG] sb-* cookies: [...]               ← 空数组 = 无 auth cookie
[DEBUG] response status: 401/500 body: {...}
```

### Edge Function 日志

```bash
# 实时查看日志
supabase functions logs gemini-proxy --tail

# 查看最近日志
supabase functions logs gemini-proxy
```

### 数据库检查

```bash
# 通过 REST API 检查（不需要直连数据库）
SERVICE_KEY="<service_role_key>"
REF="<project-ref>"

# 检查 member_profile
curl -s "https://$REF.supabase.co/rest/v1/member_profile?limit=5" \
  -H "apikey: $SERVICE_KEY" -H "Authorization: Bearer $SERVICE_KEY"

# 检查 billing_wallet
curl -s "https://$REF.supabase.co/rest/v1/billing_wallet?limit=5" \
  -H "apikey: $SERVICE_KEY" -H "Authorization: Bearer $SERVICE_KEY"

# 检查 gen_log（最近的 AI 调用）
curl -s "https://$REF.supabase.co/rest/v1/gen_log?order=created_at.desc&limit=5" \
  -H "apikey: $SERVICE_KEY" -H "Authorization: Bearer $SERVICE_KEY"

# 测试 RPC 函数
curl -s "https://$REF.supabase.co/rest/v1/rpc/billing_deduct_credits" \
  -H "apikey: $SERVICE_KEY" -H "Authorization: Bearer $SERVICE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"p_user_id":"<uid>","p_amount":0.5,"p_action":"chat","p_model":"gemini-3-flash-preview"}'
```

### 网络层诊断

如果 `supabase db push` 因 TLS/超时失败（常见于 VPN 或受限网络）：
1. DNS 解析到 `198.18.x.x`（Tailscale/VPN 代理地址）= 网络被拦截
2. 备选方案：通过 Supabase Dashboard SQL Editor 手动执行迁移 SQL
3. 或尝试 pooler 连接：`postgresql://postgres.<ref>:<password>@aws-0-<region>.pooler.supabase.com:6543/postgres`

---

## Related Skills

| Skill | 何时使用 |
|-------|---------|
| **mvp-billing-system** | 生产级计费系统完整实施手册（EN），含本 skill 排查的 Edge Function 背后的业务逻辑 |
| **mvp-billing-system-cn** | 同上，中文版 |
| **supabase-developer** | Supabase 全功能参考（Auth、Database、Storage、Real-time） |
| **stripe-payments** | Stripe 支付集成入门版（Next.js Route Handler 模式） |
| **deploy-gate** | 部署前六关门禁检查 |
