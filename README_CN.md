<div align="center">

# SaaS Skills Suite

**经过生产验证的 Claude Code 技能套件，从零构建 SaaS 产品**

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
> **什么是 Claude Code Skill？** Skill 是放在 `~/.claude/skills/` 目录下的 Markdown 指令文件，教会 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) *如何*完成特定任务。当你提到关键词（如「计费系统」「部署」），Claude Code 会自动加载匹配的 Skill，遵循其中的模式、代码模板和最佳实践。可以理解为**可复用的专家知识**，让 Claude Code 变成某个领域的专家。

## ⚡ 项目概述

**SaaS Skills Suite** 是一套 10 个 Claude Code 技能，覆盖 **SaaS 开发全生命周期** 及文档标准化 —— 从项目骨架搭建到生产环境部署门禁检查。

所有技能均提取自一个真实的生产项目（[InspirationLab Online](https://www.inspirationlab.net)），使用 Next.js + Supabase + Stripe 在 4 天内完成 32 次提交。每个「踩坑」章节都是我们在生产环境中实际遇到的 Bug。每个代码模板都可以直接复制使用。

相比通用的 AI 编程辅助，本套件拥有 🚀 **六大优势**：

1. **实战验证，非纸上谈兵**：每个代码模板、每个踩坑记录、每条调试技巧都来自真实产品的上线过程。不是教程 —— 而是一本战地日记。

2. **全生命周期覆盖**：从 `project-scaffold`（第 0 天架构设计）到 `mvp-billing-system`（第 2-3 天计费系统）再到 `deploy-gate`（第 4 天上线检查）—— 技能之间无缝衔接，没有断层。

3. **深度计费专长**：1,300+ 行专注于 SaaS 最难的部分 —— 支付。双桶钱包、双轨支付、积分幂等、Stripe API 版本地狱、Webhook 去重。这些需要数周调试的问题，在这里只需几分钟就能找到答案。

4. **互联互通的技能图谱**：每个 Skill 底部都有「Related Skills」表格。Claude Code 始终知道下一步该加载哪个技能。没有死胡同，没有孤立的知识点。

5. **双语支持（英文 + 中文）**：核心计费技能提供英文和中文两个版本。选择最符合你思维习惯的语言。

6. **零锁定**：纯 Markdown 文件。随意编辑、Fork、替换代码模板。适用于任何 Claude Code 环境，无需插件或扩展。

> 这些技能针对 **Next.js + Supabase + Stripe** 设计，但架构模式（幂等、Webhook 处理、积分系统、部署门禁）可迁移到任何技术栈。

## 📊 输入 → 输出：这套技能改变了什么

```
┌─ 输入状态（Day 0）─────────────────────────┐
│                                            │
│  · 一个产品想法                             │
│  · 一台装了 Node.js 的电脑                  │
│  · GitHub / Supabase / Stripe 账号          │
│  · 零代码，零文档，零架构                    │
│                                            │
└────────────────┬───────────────────────────┘
                 │
            10 个 Skill
                 │
┌─ 输出状态（Day 4）─────────────────────────┐
│                                            │
│  · 8 份架构文档 + 约束代码骨架               │
│  · 全栈 Next.js 应用（SSR + Auth + RLS）    │
│  · 生产级计费系统（双钱包 + 双轨支付）        │
│  · 5 个 Edge Function 已部署                │
│  · 6 关部署门禁全部通过                      │
│  · 双语专业 README                          │
│  · 可上线的 SaaS 产品                       │
│                                            │
└────────────────────────────────────────────┘
```

### 逐层状态流转

| # | Skill | 输入状态 | 输出状态 | 关键产物 |
|:-:|-------|---------|---------|---------|
| 1 | **project-scaffold** | 产品想法 + 模糊需求 | 8 份耦合架构文档 + 约束代码骨架 | `docs/*.md`, `CLAUDE.md`, `src/types/*.ts`, `scripts/` |
| 2 | **saas-quickstart** | GitHub / Supabase / Stripe 账号 | 可运行的 starter kit + 配置完毕的服务 | `.env.local`, Stripe fixtures, Vercel 部署 |
| 3 | **nextjs-fullstack** | 空的 Next.js 项目 | 全栈代码结构 + 认证中间件 + Server Actions | `app/`, `features/`, `middleware.ts`, `lib/` |
| 4 | **supabase-developer** | 空的 Supabase 项目 | Auth + RLS + Storage + Real-time + Edge Functions 就绪 | `migrations/`, RLS 策略, Storage buckets |
| 5 | **stripe-payments** | Next.js + Supabase + Stripe 账号 | 简单订阅流程跑通（Checkout → Webhook → 门控） | `api/webhooks/`, `lib/stripe.ts`, subscriptions 表 |
| 6 | **mvp-billing-system** | 基础 Stripe 集成完成 | 生产级计费（双钱包、幂等、限流、审计） | 5 张表, 7 个 RPC, 5 个 Edge Function, 计费 UI |
| 7 | **mvp-billing-system-cn** | 同上 | 同上（中文版） | 同上 |
| 8 | **supabase-gemini-deploy** | Edge Function 部署失败（401/500/CORS） | 所有 Edge Function 正常运行 | 诊断报告, 修复后配置, 调试工具集 |
| 9 | **deploy-gate** | 代码完成，准备上线 | 6 关门禁报告（通过/阻断 + 原因） | 编译结果, 环境审计, Git 快照, 最终报告 |
| 10 | **readme-standard** | 项目名 + 功能列表 + 技术栈 | 专业双语 README | `README.md`, `README_CN.md`, `LICENSE` |

### 能力累积链

```
想法
 │
 ├─→ [project-scaffold]       → +架构文档 +约束代码
 │                                    │
 ├─→ [saas-quickstart]         → +可运行项目 +服务账号
 │                                    │
 │   ┌────────────────────────────────┤
 │   │              │                 │
 │   ▼              ▼                 ▼
 │ nextjs-       supabase-       stripe-
 │ fullstack     developer       payments
 │ → +SSR         → +DB +RLS      → +Checkout
 │ → +Auth中间件   → +Storage      → +Webhook
 │ → +Actions     → +Real-time    → +订阅门控
 │   │              │                 │
 │   └──────────────┼─────────────────┘
 │                  │
 │                  ▼
 │   [mvp-billing-system]      → +双钱包 +双轨支付
 │                                +幂等 +限流
 │                                +监控 +8个踩坑
 │                  │
 │   [supabase-gemini-deploy]  → +Edge Function 全部正常
 │                  │
 │                  ▼
 │   [deploy-gate]             → +6关通过 = 可上线
 │                  │
 │                  ▼
 └─→ [readme-standard]        → +专业README = 可开源
                    │
                    ▼
             上线的 SaaS 产品
```

> **一句话总结**：输入是一个想法 + 三个账号（GitHub, Supabase, Stripe）。输出是一个有计费、有认证、有文档、经过 6 关门禁检验的可上线 SaaS 产品。

## 📦 快速安装

### 安装全部（推荐）

```bash
git clone https://github.com/Alex647648/saas-skills-suite.git
cp -r saas-skills-suite/*/ ~/.claude/skills/
```

### 按需安装

```bash
git clone https://github.com/Alex647648/saas-skills-suite.git

# 示例：只安装计费（中文版）+ 部署门禁
cp -r saas-skills-suite/mvp-billing-system-cn ~/.claude/skills/
cp -r saas-skills-suite/deploy-gate ~/.claude/skills/
```

### 验证安装

```bash
# 打开 Claude Code，输入：
/skills
# 你应该能看到已安装的技能列表

# 试试说：
"帮我搭建 Stripe 支付"
# → stripe-payments 技能自动激活
```

## 🏗️ 系统架构

### 技能依赖图

9 个技能形成分层架构，上层依赖下层提供的上下文：

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│    project-scaffold              "新建项目 / 搭骨架"     │
│    ┌──────────────────────────┐                         │
│    │ 8 份架构文档              │                         │
│    │ + 约束代码骨架            │                         │
│    └────────────┬─────────────┘                         │
│                 │                                       │
│    saas-quickstart               "快速上线"              │
│    ┌────────────┴─────────────┐                         │
│    │ 5 步启动指南              │                         │
│    └────────────┬─────────────┘                         │
│                 │                                       │
│    ┌────────────┼─────────────────────┐                 │
│    │            │                     │                 │
│    ▼            ▼                     ▼                 │
│  nextjs-     supabase-          stripe-                 │
│  fullstack   developer          payments                │
│  (141 行)    (1,492 行)         (224 行)                │
│    │            │                     │                 │
│    └────────────┼─────────────────────┘                 │
│                 │                                       │
│    ┌────────────┴─────────────────────┐                 │
│    │                                  │                 │
│    ▼                                  ▼                 │
│  mvp-billing-system            supabase-gemini-         │
│  (1,342 行 英文)               deploy                   │
│  (1,335 行 中文)               (522 行)                 │
│    │                                  │                 │
│    └────────────┬─────────────────────┘                 │
│                 │                                       │
│                 ▼                                       │
│           deploy-gate                "能不能上线？"      │
│           (100 行)                                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 技能速查表

| # | 技能 | 行数 | 触发关键词 | 功能 |
|:-:|------|-----:|-----------|------|
| 1 | **project-scaffold** | 139 | 「新项目」「搭骨架」「初始化」 | 生成 8 份耦合架构文档 + TypeScript 约束代码骨架 |
| 2 | **saas-quickstart** | 71 | 「新建SaaS」「快速上线」「模板」 | 从 starter kit 克隆到部署的 5 步指南 |
| 3 | **nextjs-fullstack** | 141 | 「Next.js」「路由」「Server Action」 | App Router 模式：Server Components、Actions、中间件、缓存 |
| 4 | **supabase-developer** | 1,492 | 「Supabase」「数据库」「认证」「Edge Function」 | 全功能参考：Auth（`@supabase/ssr`）、DB、存储、实时、Edge Functions |
| 5 | **stripe-payments** | 224 | 「Stripe入门」「支付入门」 | 入门级 Stripe：Checkout、Webhook、订阅门控 |
| 6 | **mvp-billing-system** | 1,342 | "billing system", "credits", "subscription" | 生产级计费手册（英文）：双钱包、8个踩坑、测试、监控 |
| 7 | **mvp-billing-system-cn** | 1,335 | 「计费系统」「支付」「订阅」 | 同上，全中文版本 |
| 8 | **supabase-gemini-deploy** | 522 | 「Edge Function报错」「401」「CORS」 | 部署排查：JWT（ES256/HS256）、CORS、限流 |
| 9 | **deploy-gate** | 100 | 「部署」「发布」「上线」 | 6 关顺序质量门禁 |
| 10 | **readme-standard** | 406 | 「写README」「项目文档」「优化介绍」 | 标准化 README 写作指南，含徽章、结构、双语模板 |

## 🔍 技能详解

### 1. project-scaffold（项目骨架）

> **4 阶段工作流**，生成完整项目骨架

| 阶段 | 操作 | 产出 |
|:----:|------|------|
| 1 | 对话式收集需求 | 项目简报 |
| 2 | 生成 8 份架构文档 | 验证目标、数据模型、API 契约、设计系统、Agent 规范、开发约束、任务看板、部署清单 |
| 3 | 生成约束代码 | Branded 类型、状态机、确认拦截中间件、ESLint 规则、lint 脚本 |
| 4 | 一致性校验 | 跨文档一致性检查通过 |

包含 `references/templates/` 目录下的文档和代码模板。

---

### 2. saas-quickstart（快速启动）

> **从零到上线的最短路径**

| 步骤 | 内容 | 时间 |
|:----:|------|:----:|
| 1 | 克隆 starter kit | 10 分钟 |
| 2 | 配置 Supabase + Stripe | 30 分钟 |
| 3 | 开发你的核心功能 | 1-3 天 |
| 4 | 添加支付门控 | 30 分钟 |
| 5 | 部署到 Vercel | 10 分钟 |

---

### 3. nextjs-fullstack（全栈开发）

> **Next.js 14/15+ App Router** 模式参考

- Server Components vs Client Components —— 何时用 `'use client'`
- Server Actions 做数据变更，含 Zod 校验
- Route Handlers 处理外部 Webhook
- Middleware 做认证保护
- 性能目标：TTFB < 200ms, LCP < 2.5s

---

### 4. supabase-developer（全功能参考）

> **最全面的技能** —— 1,492 行覆盖 Supabase 全部功能

| 模块 | 核心内容 |
|------|---------|
| **Auth** | `@supabase/ssr` 配置、`getUser()` vs `getSession()`、OAuth、中间件（已从废弃的 `auth-helpers` 更新） |
| **数据库** | Schema 设计、RLS 策略、迁移、RPC 函数、`FOR UPDATE` 行锁 |
| **存储** | Bucket、策略、签名 URL、图片变换 |
| **实时** | Channel、Presence、Broadcast |
| **Edge Functions** | Deno 运行时、CORS、共享模块、`--no-verify-jwt` 部署 |

---

### 5. stripe-payments（支付入门）

> **入门级** Stripe 集成，使用 Next.js Route Handlers

覆盖 Checkout Session 创建、Webhook 处理、订阅同步、支付门控。

> **注意**：如需生产级计费（积分、双钱包、幂等），请使用 **mvp-billing-system-cn**。

---

### 6. mvp-billing-system（生产级计费）

> **核心技能** —— 1,342 行。生产级 SaaS 计费完整手册。

| 阶段 | 内容 |
|:----:|------|
| 1 | 数据库 Schema —— 5 张核心表、RLS、触发器、索引 |
| 2 | 计费 RPC 函数 —— `FOR UPDATE` 原子扣减、幂等发放 |
| 3 | Edge Functions —— Stripe 结账（3 种流程）、Webhook 处理、订阅管理 |
| 4 | 前端组件 —— 定价面板、付费墙弹窗、账单历史 |
| 5 | 进阶 —— 双轨支付（自动续费 + 手动）+ 双桶钱包（订阅 + 加购） |
| 6 | 安全加固 —— 数据库限流、审计日志、pg_cron 清理 |

**额外章节，帮你节省数周调试时间**：

| 章节 | 你能获得的 |
|------|-----------|
| 8 个踩坑 | Stripe API 版本地狱、JWT ES256/HS256、重复订阅、Webhook 版本不匹配等 |
| 幂等模式 | Webhook 去重表、积分发放复合键、退款检查、加购去重 |
| 测试 & QA | Stripe CLI Webhook 转发、Test Clocks、20+ 测试场景清单 |
| 监控 | 结构化 JSON 日志、5 条健康检查 SQL、告警策略表 |
| 调试速查 | 症状 → 原因 → 修复 对照表（10 个常见问题） |

---

### 7. mvp-billing-system-cn（中文版）

与 `mvp-billing-system` 内容完全相同，全部用中文编写。根据你的偏好选择。

---

### 8. supabase-gemini-deploy（部署排查）

> **排查手册**，按决策树组织 —— 症状 → 诊断 → 修复

- JWT 问题（ES256 vs HS256 不匹配）
- CORS 配置
- API Key 和环境变量配置
- 计费 RPC 调试
- 速率限制
- 网络诊断（VPN/TLS）

---

### 9. deploy-gate（部署门禁）

> **6 道顺序关卡** —— 全部通过才放行

| 关卡 | 检查内容 | 自动 |
|:----:|---------|:----:|
| 1 | 编译 + TypeScript + ESLint | 是 |
| 2 | 约束脚本（不存在时优雅跳过） | 是 |
| 3 | 环境变量 + 迁移文件 | 是 |
| 4 | 功能回归清单 | 手动 |
| 5 | Git 状态 + 分支 + CHANGELOG | 是 |
| 6 | 最终汇总报告 | — |

---

### 10. readme-standard（README 写作标准）

> **标准化 GitHub README 写作指南** —— 本 README 就是用这个技能写的

一套完整的专业项目文档模板系统：

- **9 节标准结构**：头部、提示框、概述、快速安装、架构、详细介绍、技术栈、自定义、FAQ、许可证
- **Shields.io 徽章模板库**：社区指标、项目元数据、技术栈、状态徽章 —— 统一 `flat-square` 风格
- **Emoji 章节标题速查**：13 个标准 emoji 对应章节映射
- **架构图规则**：Box-drawing 字符、70 列宽度限制、数据流标注
- **双语规范**：文件命名、结构对齐、中英文间距规则
- **反面模式**：9 个常见 README 错误及避免方法

## ⚙️ 技术栈

这些技能针对以下技术栈设计，但很多模式可迁移到其他栈：

| 层 | 技术 |
|----|------|
| 语言 | TypeScript |
| 前端 | Next.js 14/15+（App Router） |
| 后端 | Supabase Edge Functions (Deno) + Next.js API Routes |
| 数据库 | PostgreSQL（Supabase 托管） |
| 认证 | Supabase Auth（`@supabase/ssr`） |
| 支付 | Stripe（Checkout、Subscriptions、Webhooks） |
| 部署 | Vercel + Supabase |

## 🔧 自定义

每个技能都是独立的 Markdown 文件，修改方式：

1. **编辑 `SKILL.md`** —— 修改代码模板，加入你自己的模式
2. **调整触发词** —— 修改 YAML 头部的 `description` 字段
3. **加入项目信息** —— 插入你的 price ID、表名、API 端点
4. **删除不需要的** —— 每个技能独立工作，互不依赖

### 文件结构

```
~/.claude/skills/
├── project-scaffold/
│   ├── SKILL.md                             # 骨架生成器技能
│   └── references/templates/                # 文档 + 代码模板
│       ├── constraint-code.md               # TypeScript 约束模板
│       ├── doc-templates.md                 # 8 份架构文档模板
│       └── validation-target.md             # 验证目标模板
├── saas-quickstart/
│   └── SKILL.md                             # 5 步启动指南
├── nextjs-fullstack/
│   └── SKILL.md                             # App Router 模式
├── supabase-developer/
│   └── SKILL.md                             # Supabase 全功能参考
├── stripe-payments/
│   └── SKILL.md                             # Stripe 入门指南
├── mvp-billing-system/
│   └── SKILL.md                             # 生产级计费（英文）
├── mvp-billing-system-cn/
│   └── SKILL.md                             # 生产级计费（中文）
├── supabase-gemini-deploy/
│   └── SKILL.md                             # Edge Function 排查
├── deploy-gate/
│   └── SKILL.md                             # 6 关部署门禁
└── readme-standard/
    └── SKILL.md                             # README 写作标准
```

## ❓ 常见问题

**Q: 必须全部安装吗？**
> 不用。每个技能独立工作。从你当前需要的开始。`mvp-billing-system-cn` + `deploy-gate` 组合覆盖面最广。

**Q: 会和其他 Claude Code 技能冲突吗？**
> 不会。技能按关键词按需加载，互不干扰，也不影响其他来源的技能。

**Q: 能用在其他技术栈上吗？**
> 架构模式（幂等、Webhook 处理、积分系统）与技术栈无关。代码模板针对 Next.js + Supabase + Stripe，但可以适配。

**Q: 怎么更新？**
> ```bash
> cd saas-skills-suite && git pull
> cp -r */* ~/.claude/skills/
> ```

**Q: 发现问题 / 想贡献代码？**
> 欢迎提 [Issue](https://github.com/Alex647648/saas-skills-suite/issues) 或 [Pull Request](https://github.com/Alex647648/saas-skills-suite/pulls)！

## 📄 许可证

[MIT](./LICENSE) —— 自由使用，自由修改。不强制署名，但如果觉得有用，欢迎点个 Star 或保留出处链接。
