# 约束代码模板

> 由 project-scaffold 阶段三生成。`{{ }}` 根据项目文档填充。
> 所有代码纯 TypeScript，不依赖特定框架。路径根据技术选型调整。

---

## 1. src/types/provider.ts

```typescript
// ================================================
// 📋 约束清单（此文件相关）：
// 1. 这是 Provider 的唯一定义位置（DATA_MODEL §2.2）
// 2. 新增 provider 只改此文件，Record 强制补全所有配置
// 3. 不可在其他文件中硬编码 provider 字符串
// ================================================

export const PROVIDERS = [{{ 'sora', 'vidu', 'jimeng' 等 }}] as const;
export type Provider = typeof PROVIDERS[number];

export const PROVIDER_CONFIG: Record<Provider, {
  displayName: string;
  maxTimeoutMs: number;
  creditsPerTask: number;
  requiresConfirmation: boolean;
}> = {
  // {{ 每个 provider 的配置 }}
  // 遗漏任何一个 provider，TypeScript 直接报错
};

export function isValidProvider(value: string): value is Provider {
  return (PROVIDERS as readonly string[]).includes(value);
}
```

## 2. src/types/task-status.ts

```typescript
// ================================================
// 📋 约束清单（此文件相关）：
// 1. 状态枚举与 DATA_MODEL §3.1 状态机一一对应
// 2. validateTransition 在运行时拦截非法状态跳转
// 3. assertNever 在编译时拦截遗漏新状态的 switch case
// ================================================

export const TASK_STATUSES = [
  'queued', 'processing', 'completed', 'failed', 'cancelled'
] as const;
export type TaskStatus = (typeof TASK_STATUSES)[number];

const VALID_TRANSITIONS: Record<TaskStatus, readonly TaskStatus[]> = {
  queued:     ['processing', 'cancelled'],
  processing: ['completed', 'failed', 'cancelled'],
  completed:  [],
  failed:     [],
  cancelled:  [],
};

export function validateTransition(from: TaskStatus, to: TaskStatus): void {
  if (!VALID_TRANSITIONS[from].includes(to)) {
    throw new Error(
      `INVALID_STATE_TRANSITION: ${from} → ${to} 不合法。` +
      `允许：${from} → [${VALID_TRANSITIONS[from].join(', ')}]\n` +
      `参见：docs/DATA_MODEL.md §3.1`
    );
  }
}

function assertNever(x: never): never {
  throw new Error(`未处理的值: ${x}`);
}

export function getStatusLabel(status: TaskStatus): string {
  switch (status) {
    case 'queued':     return '排队中';
    case 'processing': return '生成中';
    case 'completed':  return '已完成';
    case 'failed':     return '失败';
    case 'cancelled':  return '已取消';
    default: return assertNever(status);
  }
}
```

## 3. src/types/url.ts

```typescript
// ================================================
// 📋 约束清单（此文件相关）：
// 1. PersistentUrl 和 TemporaryUrl 是不同类型，编译器强制区分
// 2. 数据库持久化字段只接受 PersistentUrl
// 3. blob:/data: URL 必须先经过 ensurePersistedUrl 转换
// 来源：DATA_MODEL §3.2
// ================================================

declare const __brand_persistent: unique symbol;
declare const __brand_temporary: unique symbol;

export type PersistentUrl = string & { readonly [__brand_persistent]: true };
export type TemporaryUrl = string & { readonly [__brand_temporary]: true };
export type AnyUrl = PersistentUrl | TemporaryUrl;

const TEMP_URL_PATTERNS = [/^blob:/, /^data:/, /^\/tmp\//];

export function isTemporaryUrl(url: string): boolean {
  return TEMP_URL_PATTERNS.some(p => p.test(url));
}

export function asPersistentUrl(url: string): PersistentUrl {
  if (isTemporaryUrl(url)) {
    throw new Error(
      `CONSTRAINT_VIOLATION: "${url.slice(0, 40)}..." 是临时URL，` +
      `不能用于持久化存储。请先调用 ensurePersistedUrl()。\n` +
      `参见：docs/DATA_MODEL.md §3.2`
    );
  }
  return url as PersistentUrl;
}

export function asTemporaryUrl(url: string): TemporaryUrl {
  return url as TemporaryUrl;
}

/**
 * 将任意 URL 转为持久化 URL。
 * ⚠️ 需要接入项目的对象存储上传逻辑。
 */
export async function ensurePersistedUrl(
  url: string,
  context: { folder: string; filename?: string }
): Promise<PersistentUrl> {
  if (!isTemporaryUrl(url)) {
    return url as PersistentUrl;
  }
  // TODO: 替换为项目实际的上传逻辑（R2/S3/OSS）
  // const uploaded = await storage.upload(url, context);
  // return uploaded as PersistentUrl;
  throw new Error(
    'ensurePersistedUrl: 需要实现对象存储上传。\n' +
    '请在此函数中接入 R2/S3/OSS 的上传逻辑。'
  );
}
```

## 4. src/types/api-response.ts

```typescript
// ================================================
// 📋 约束清单（此文件相关）：
// 1. ErrorCode 枚举与 API_CONTRACT §1.1 错误码表一一对应
// 2. 新增错误码必须同步更新 docs/API_CONTRACT.md §1.1
// 3. 所有 API 端点必须使用 ApiResponse 类型
// ================================================

export const ERROR_CODES = [
  'UNAUTHORIZED',
  'FORBIDDEN',
  'NOT_FOUND',
  'INSUFFICIENT_CREDITS',
  'PROVIDER_ERROR',
  'RATE_LIMITED',
  'TIMEOUT',
  'VALIDATION_ERROR',
  'CONFIRMATION_REQUIRED',
] as const;

export type ErrorCode = (typeof ERROR_CODES)[number];

export interface ApiSuccess<T> {
  success: true;
  data: T;
  metadata?: { timestamp: string; requestId: string };
}

export interface ApiError {
  success: false;
  error: { code: ErrorCode; message: string; details?: unknown };
}

export type ApiResponse<T> = ApiSuccess<T> | ApiError;
```

## 5. src/lib/api-response.ts

```typescript
// ================================================
// 📋 约束清单（此文件相关）：
// 1. 所有端点必须用 apiSuccess/apiError，禁止手工构造响应
// 2. 错误码必须来自 ErrorCode 枚举
// 来源：CLAUDE.md §4.1
// ================================================

import type { ErrorCode, ApiSuccess, ApiError } from '../types/api-response';

export function apiSuccess<T>(data: T): ApiSuccess<T> {
  return {
    success: true,
    data,
    metadata: {
      timestamp: new Date().toISOString(),
      requestId: crypto.randomUUID?.() ?? Math.random().toString(36),
    },
  };
}

export function apiError(code: ErrorCode, message: string, details?: unknown): ApiError {
  return {
    success: false,
    error: { code, message, details },
  };
}
```

## 6. src/lib/data-service.ts

```typescript
// ================================================
// 📋 约束清单（此文件相关）：
// 1. 这是数据操作的唯一入口（CLAUDE §3）
// 2. 其他文件禁止直接引用数据库客户端（ESLint 强制）
// 3. 所有写入操作自动检查临时 URL（DATA_MODEL §3.2）
// ================================================

import { isTemporaryUrl } from '../types/url';

/**
 * 写入前自动检查所有字符串字段是否包含临时 URL。
 * allowedTempFields 中列出的字段可以暂存临时 URL（如 output_url）。
 */
function guardTemporaryUrls(
  data: Record<string, unknown>,
  allowedTempFields: string[] = []
): void {
  for (const [key, value] of Object.entries(data)) {
    if (typeof value !== 'string') continue;
    if (allowedTempFields.includes(key)) continue;
    if (isTemporaryUrl(value)) {
      throw new Error(
        `TEMPORARY_URL_VIOLATION: 字段 "${key}" 包含临时URL。` +
        `请先调用 ensurePersistedUrl() 转换。\n` +
        `允许临时URL的字段: [${allowedTempFields.join(', ')}]\n` +
        `参见：docs/DATA_MODEL.md §3.2`
      );
    }
  }
}

/**
 * 数据服务基类。项目需要继承此类并接入实际数据库。
 * 继承后覆盖 _insert / _update / _delete / _query 方法。
 */
export abstract class DataServiceBase {
  /** 插入数据（自动检查临时URL） */
  async insert(table: string, data: Record<string, unknown>): Promise<unknown> {
    guardTemporaryUrls(data, ['output_url']);
    return this._insert(table, data);
  }

  /** 更新数据（自动检查临时URL） */
  async update(
    table: string,
    id: string,
    data: Record<string, unknown>
  ): Promise<unknown> {
    guardTemporaryUrls(data, ['output_url']);
    return this._update(table, id, data);
  }

  // 子类实现实际的数据库操作
  protected abstract _insert(table: string, data: Record<string, unknown>): Promise<unknown>;
  protected abstract _update(table: string, id: string, data: Record<string, unknown>): Promise<unknown>;
  protected abstract _delete(table: string, id: string): Promise<unknown>;
  protected abstract _query(table: string, filters: Record<string, unknown>): Promise<unknown[]>;
}
```

## 7. src/middleware/confirm-guard.ts

```typescript
// ================================================
// 📋 约束清单（此文件相关）：
// 1. 需确认端点列表与 API_CONTRACT "确认:✅" 的端点一致
// 2. 此中间件在路由层拦截，业务代码无法绕过
// 3. 新增需确认端点时，必须同步更新此文件和 API_CONTRACT §3
// 来源：AGENTS §2.1 + API_CONTRACT
// ================================================

import { PROVIDER_CONFIG, type Provider } from '../types/provider';

/** 需要用户确认才能执行的端点 */
const CONFIRMATION_REQUIRED = new Set<string>([
  {{ 从 API_CONTRACT 中提取，如 'POST /api/tasks/generate' }}
]);

export function isConfirmationRequired(method: string, path: string): boolean {
  return CONFIRMATION_REQUIRED.has(`${method.toUpperCase()} ${path}`);
}

export interface ConfirmationCheck {
  confirmed: boolean;
  estimatedCredits?: number;
  actionSummary?: string;
}

export function checkConfirmation(
  method: string,
  path: string,
  body: Record<string, unknown> = {},
  headers: Record<string, string> = {}
): ConfirmationCheck {
  if (!isConfirmationRequired(method, path)) {
    return { confirmed: true };
  }
  const confirmed =
    headers['x-user-confirmed'] === 'true' || body.user_confirmed === true;
  if (confirmed) return { confirmed: true };

  const provider = body.provider as Provider | undefined;
  return {
    confirmed: false,
    estimatedCredits: provider ? PROVIDER_CONFIG[provider]?.creditsPerTask : undefined,
    actionSummary: `${method.toUpperCase()} ${path}`,
  };
}
```

## 8. scripts/lint-constraints.js

```javascript
#!/usr/bin/env node
// ================================================
// 自定义约束扫描。pre-commit 或 CI 中运行。
// 每条错误含文档引用，AI 看到后能自行修复。
// 用法: node scripts/lint-constraints.js [src目录]
// ================================================

const fs = require('fs');
const path = require('path');

function walk(dir, exts) {
  const results = [];
  if (!fs.existsSync(dir)) return results;
  for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
    const full = path.join(dir, entry.name);
    if (entry.isDirectory()) {
      if (['node_modules', '.next', '.git', 'dist', '.nuxt'].includes(entry.name)) continue;
      results.push(...walk(full, exts));
    } else if (exts.some(e => entry.name.endsWith(e))) {
      results.push(full);
    }
  }
  return results;
}

const V = [];  // violations
const src = path.resolve(process.argv[2] || 'src');

// R1: 禁止硬编码颜色 → DESIGN_SYSTEM §2.2
for (const f of walk(src, ['.tsx', '.jsx'])) {
  const c = fs.readFileSync(f, 'utf-8');
  const m = c.match(/className="[^"]*(?:bg|text|border)-\[#[0-9a-fA-F]{3,8}\]/g);
  if (m) V.push({ f, r: 'no-hardcoded-colors', m: m.join(', '), d: 'DESIGN_SYSTEM §2.2' });
}

// R2: 禁止厚重阴影 → DESIGN_SYSTEM §7
for (const f of walk(src, ['.tsx', '.jsx'])) {
  const c = fs.readFileSync(f, 'utf-8');
  const m = c.match(/shadow-(?:lg|xl|2xl)/g);
  if (m) V.push({ f, r: 'no-heavy-shadows', m: m.join(', '), d: 'DESIGN_SYSTEM §7' });
}

// R3: z-index 超出范围 → DESIGN_SYSTEM §6
for (const f of walk(src, ['.tsx', '.jsx'])) {
  const c = fs.readFileSync(f, 'utf-8');
  for (const [, val] of c.matchAll(/z-\[(\d+)\]/g)) {
    if (parseInt(val) > 49) V.push({ f, r: 'z-index-range', m: `z-[${val}] > 49`, d: 'DESIGN_SYSTEM §6' });
  }
}

// R4: 禁止直接引用数据库客户端 → CLAUDE §3
for (const f of walk(src, ['.ts', '.tsx'])) {
  if (f.includes('data-service') || f.includes('dataService') || f.includes('supabase/')) continue;
  const c = fs.readFileSync(f, 'utf-8');
  if (/from\s+['"].*supabase/.test(c) || /from\s+['"].*prisma/.test(c) || /from\s+['"].*drizzle/.test(c)) {
    V.push({ f, r: 'no-direct-db', m: '禁止直接引用数据库客户端，请用 data-service', d: 'CLAUDE §3' });
  }
}

// R5: API路由文件缺显式超时（Next.js/Vercel 项目适用）→ CLAUDE §4.3
for (const f of walk(src, ['.ts'])) {
  if (!f.includes('route.ts') && !f.includes('route.js')) continue;
  const c = fs.readFileSync(f, 'utf-8');
  if (!c.includes('maxDuration') && !c.includes('MAX_DURATION')) {
    V.push({ f, r: 'explicit-timeout', m: '缺少显式超时配置', d: 'CLAUDE §4.3 + API_CONTRACT' });
  }
}

// 输出
if (V.length) {
  console.error(`\n❌ ${V.length} 个约束违规:\n`);
  for (const v of V) {
    console.error(`  ${path.relative(process.cwd(), v.f)}`);
    console.error(`    [${v.r}] ${v.m}`);
    console.error(`    参见: ${v.d}\n`);
  }
  process.exit(1);
} else {
  console.log('✅ 约束检查通过');
}
```

## 9. .eslintrc.js 补充规则

```javascript
// 追加到项目 ESLint 配置中：
{
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [{
        group: ['**/supabase/client*', '**/prisma/client*', '**/drizzle*'],
        message: '❌ 禁止直接引用数据库客户端。请使用 data-service。参见 CLAUDE.md §3',
      }],
    }],
  },
  overrides: [{
    files: ['**/data-service.*', '**/dataService.*', '**/supabase/**', '**/db/**'],
    rules: { 'no-restricted-imports': 'off' },
  }],
}
```
