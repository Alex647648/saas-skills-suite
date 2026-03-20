# SaaS Skills Suite

**[English README](./README.md)**

> 9 个经过生产验证的 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 技能，覆盖 SaaS 开发全生命周期 —— 从项目骨架搭建到部署门禁检查。

源自一个真实 SaaS 产品（Next.js + Supabase + Stripe）4 天 32 次提交的完整开发历程。每个「踩坑」章节都是我们在生产环境中实际遇到的 Bug。

---

## 什么是 Claude Code Skill？

Skill 是 Markdown 格式的指令文件，教会 Claude Code *如何*完成特定任务。当你提到关键词（如「计费系统」、「部署」），Claude Code 会自动加载匹配的 Skill，遵循其中的模式、代码模板和最佳实践。

可以理解为**可复用的专家知识**，让 Claude Code 变成某个领域的专家。

---

## 适合谁用？

- **独立开发者**：用 Next.js + Supabase + Stripe 构建 SaaS 产品
- **小团队**：希望在 AI 辅助开发中保持架构决策一致性
- **所有人**：想跳过 SaaS 计费、认证、部署的「摸索期」

---

## 快速安装

### 方式一：安装全部（推荐）

```bash
git clone https://github.com/Alex647648/saas-skills-suite.git
cp -r saas-skills-suite/*/ ~/.claude/skills/
```

### 方式二：按需安装

```bash
git clone https://github.com/Alex647648/saas-skills-suite.git

# 示例：只安装计费 + 部署门禁
cp -r saas-skills-suite/mvp-billing-system-cn ~/.claude/skills/
cp -r saas-skills-suite/deploy-gate ~/.claude/skills/
```

### 验证安装

打开 Claude Code，输入：

```
/skills
```

你应该能看到已安装的 Skill 列表。试着说「帮我搭建 Stripe 支付」—— 相关 Skill 会自动激活。

---

## Skill 一览

| # | Skill | 行数 | 功能 |
|---|-------|-----:|------|
| 1 | [project-scaffold](#1-project-scaffold-项目骨架) | 139 | 生成 8 份架构文档 + 约束代码骨架 |
| 2 | [saas-quickstart](#2-saas-quickstart-快速启动) | 71 | 从 starter kit 到上线的 5 步指南 |
| 3 | [nextjs-fullstack](#3-nextjs-fullstack-全栈开发) | 141 | Next.js App Router 模式（Server Components、Actions、中间件） |
| 4 | [supabase-developer](#4-supabase-developer-全功能参考) | 1,492 | Supabase 全功能参考（Auth、数据库、存储、实时、Edge Functions） |
| 5 | [stripe-payments](#5-stripe-payments-支付入门) | 224 | Stripe 支付集成入门（Checkout、Webhook、订阅门控） |
| 6 | [mvp-billing-system](#6-mvp-billing-system-生产级计费) | 1,342 | 生产级计费系统完整手册（英文版） |
| 7 | [mvp-billing-system-cn](#7-mvp-billing-system-cn-中文版) | 1,335 | 同上（中文版） |
| 8 | [supabase-gemini-deploy](#8-supabase-gemini-deploy-部署排查) | 522 | Edge Function 部署排查手册 |
| 9 | [deploy-gate](#9-deploy-gate-部署门禁) | 100 | 部署前 6 关质量检查 |

**合计：~5,366 行结构化知识 + 771 行模板文件**

---

## Skill 关系图

```
                    +-----------------------+
                    |   project-scaffold    |  "新建项目 / 搭骨架"
                    |  （架构文档 +          |
                    |    约束代码骨架）       |
                    +-----------+-----------+
                                |
                    +-----------v-----------+
                    |    saas-quickstart    |  "快速上线"
                    +-----------+-----------+
                                |
              +-----------------+-----------------+
              |                 |                 |
    +---------v-------+ +------v--------+ +------v----------+
    | nextjs-fullstack| | supabase-     | | stripe-payments |
    | (App Router,    | | developer     | | （入门版：       |
    |  SSR, Actions)  | | (Auth, 数据库, | |  Checkout,      |
    |                 | |  存储, 实时)   | |  Webhook）      |
    +--------+--------+ +------+--------+ +------+----------+
              |                 |                 |
              +-----------------+-----------------+
                                |
              +-----------------+-----------------+
              |                                   |
    +---------v-----------+          +------------v-----------+
    | mvp-billing-system  |          | supabase-gemini-deploy |
    | （生产级计费：       |          | （Edge Function        |
    |  双钱包、双轨支付、  |          |  排查：JWT, CORS,      |
    |  8个踩坑、          |          |  401/500 诊断）        |
    |  测试、监控）        |          +------------+-----------+
    +---------+-----------+                       |
              |                                   |
              +-----------------------------------+
                                |
                    +-----------v-----------+
                    |      deploy-gate      |  "能不能上线？"
                    |  （6 关部署前          |
                    |    质量检查）          |
                    +-----------------------+
```

每个 Skill 底部都有 **Related Skills** 表格链接到相关 Skill，Claude Code 始终知道下一步该查哪里。

---

## 详细介绍

### 1. project-scaffold（项目骨架）

> **触发关键词**：「新项目」「搭骨架」「初始化」「项目架构」「从零开始」

4 阶段工作流，生成完整项目骨架：

1. **收集信息** — 对话式 Q&A，收集产品需求
2. **生成 8 份文档** — 验证目标、数据模型、API 契约、设计系统、Agent 规范、开发约束、任务看板、部署清单
3. **生成约束代码** — Branded 类型、状态机、确认拦截中间件、ESLint 规则、lint 脚本
4. **校验** — 运行一致性检查，确保文档间无矛盾

包含 `references/templates/` 目录下的文档和代码模板。

---

### 2. saas-quickstart（快速启动）

> **触发关键词**：「新建SaaS」「创建项目」「快速上线」「SaaS模板」

从零到上线的最短路径：

| 步骤 | 内容 | 时间 |
|------|------|------|
| 1 | 克隆 starter kit | 10 分钟 |
| 2 | 配置 Supabase + Stripe | 30 分钟 |
| 3 | 开发你的核心功能 | 1-3 天 |
| 4 | 添加支付门控 | 30 分钟 |
| 5 | 部署到 Vercel | 10 分钟 |

---

### 3. nextjs-fullstack（全栈开发）

> **触发关键词**：「Next.js」「页面」「路由」「Server Action」

覆盖 Next.js 14/15+ App Router 模式：

- Server Components vs Client Components（何时用 `'use client'`）
- Server Actions 做数据变更（含 Zod 校验）
- Route Handlers 处理 Webhook
- Middleware 做认证保护
- 性能目标（TTFB < 200ms, LCP < 2.5s）

---

### 4. supabase-developer（全功能参考）

> **触发关键词**：「Supabase」「数据库」「认证」「实时」「存储」「Edge Function」

最全面的 Skill（1,492 行），覆盖：

- **Auth**：`@supabase/ssr` 配置（已从废弃的 `auth-helpers` 更新）、`getUser()` vs `getSession()`、OAuth、中间件
- **数据库**：Schema 设计、RLS 策略、迁移、RPC 函数、`FOR UPDATE` 行锁
- **存储**：Bucket、策略、签名 URL、图片变换
- **实时**：Channel、Presence、Broadcast
- **Edge Functions**：Deno 运行时、CORS、共享模块、部署

---

### 5. stripe-payments（支付入门）

> **触发关键词**：「Stripe入门」「支付入门」「简单订阅」

入门级 Stripe 集成，使用 Next.js Route Handlers：

- Checkout Session 创建
- Webhook 事件处理
- 订阅状态同步到数据库
- 页面支付门控

**注意**：这是入门级 Skill。如需生产级计费（积分、双钱包、幂等），请使用 `mvp-billing-system-cn`。

---

### 6. mvp-billing-system（生产级计费）

> **触发关键词**：「billing system」「Stripe integration」「payment system」「credits」

核心 Skill（1,342 行），生产级 SaaS 计费完整手册：

| 阶段 | 内容 |
|------|------|
| Phase 1 | 数据库 Schema（5 张核心表、RLS、触发器、索引） |
| Phase 2 | 计费 RPC 函数（原子扣减、幂等发放） |
| Phase 3 | Edge Functions — Stripe 集成（结账、Webhook、管理） |
| Phase 4 | 前端组件（定价面板、付费墙、账单历史） |
| Phase 5 | 进阶 — 双轨支付 + 双桶钱包 |
| Phase 6 | 安全加固（限流、审计日志、pg_cron） |

额外内容：
- **8 个踩坑章节** — 真实 Bug 及修复方案（Stripe API 版本地狱、JWT ES256/HS256、重复订阅等）
- **幂等模式** — Webhook 去重、积分发放复合键、退款检查
- **测试 & QA** — Stripe CLI Webhook 转发、Test Clocks、测试场景清单
- **监控** — 结构化日志、健康检查 SQL、告警策略
- **调试速查** — 症状 → 原因 → 修复 对照表

---

### 7. mvp-billing-system-cn（中文版）

与 `mvp-billing-system` 内容完全相同，全部用中文编写。根据你的偏好选择。

---

### 8. supabase-gemini-deploy（部署排查）

> **触发关键词**：「Edge Function」「部署报错」「401」「500」「CORS」「JWT」

Supabase Edge Function 部署排查手册：

- JWT 认证问题（ES256 vs HS256 不匹配）
- CORS 配置
- API Key 和环境变量配置
- 计费 RPC 调试
- 速率限制配置
- 网络诊断（VPN/TLS 问题）

按决策树组织：症状 → 诊断 → 修复。

---

### 9. deploy-gate（部署门禁）

> **触发关键词**：「部署」「发布」「上线」「deploy」「能不能上线」

6 道顺序关卡 — 全部通过才放行：

| 关卡 | 检查内容 | 自动？ |
|------|---------|--------|
| 关1 | 编译 + TypeScript + ESLint | 是 |
| 关2 | 约束脚本（如果存在） | 是 |
| 关3 | 环境变量 + 迁移文件 | 是 |
| 关4 | 功能回归清单 | 手动 |
| 关5 | Git 状态 + 分支 + CHANGELOG | 是 |
| 关6 | 最终汇总报告 | — |

关2 在约束脚本不存在时会优雅跳过（脚本由 `project-scaffold` 生成）。

---

## 技术栈

这些 Skill 针对以下技术栈设计，但很多模式可迁移到其他栈：

| 层 | 技术 |
|----|------|
| 语言 | TypeScript |
| 前端 | Next.js 14/15+（App Router） |
| 后端 | Supabase Edge Functions (Deno) + Next.js API Routes |
| 数据库 | PostgreSQL（Supabase 托管） |
| 认证 | Supabase Auth（`@supabase/ssr`） |
| 支付 | Stripe（Checkout、Subscriptions、Webhooks） |
| 部署 | Vercel + Supabase |

---

## 自定义

每个 Skill 都是独立的 Markdown 文件，修改方式：

1. **编辑 SKILL.md** — 修改代码模板，加入你自己的模式
2. **调整触发关键词** — 修改 YAML 头部的 `description` 字段
3. **加入项目专属信息** — 你的 price ID、表名、API 端点
4. **删除不需要的** — 每个 Skill 独立工作，互不依赖

### 文件结构

```
~/.claude/skills/
├── project-scaffold/
│   ├── SKILL.md
│   └── references/templates/    # 文档 + 代码模板
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

## 常见问题

**Q: 必须全部安装吗？**
不用。每个 Skill 独立工作。从你当前需要的开始。`mvp-billing-system-cn` + `deploy-gate` 组合覆盖面最广。

**Q: 会和其他 Claude Code Skill 冲突吗？**
不会。Skill 按关键词按需加载，互不干扰，也不会影响其他来源的 Skill。

**Q: 能用在其他技术栈上吗？**
架构模式（幂等、Webhook 处理、积分系统）与技术栈无关。代码模板针对 Next.js + Supabase + Stripe，但可以适配其他栈。

**Q: 怎么更新？**
拉取最新版本并重新复制：
```bash
cd saas-skills-suite && git pull
cp -r */* ~/.claude/skills/
```

---

## 许可证

MIT
