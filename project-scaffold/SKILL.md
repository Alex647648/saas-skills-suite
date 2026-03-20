---
name: project-scaffold
description: AI原生项目骨架生成器。当用户要启动新项目、搭建骨架、初始化架构时使用。对话式收集产品信息后，一次性生成8个耦合架构文档（验证目标、数据模型、API契约、设计系统、Agent规范、开发约束、任务看板、部署清单）和TypeScript约束代码骨架（Branded类型、穷尽式状态机、确认拦截中间件、ESLint规则、lint脚本）。约束不停留在文档里，而是沉降到编译器和运行时中。只要用户提到"新项目"、"搭骨架"、"初始化"、"项目架构"、"从零开始"，立刻触发。
---

# Project Scaffold — AI 原生项目骨架生成器

> 生成前先读取 `references/templates/` 下的模板文件。
> 约束代码模板见 `references/templates/constraint-code.md`。
> 文档模板见 `references/templates/doc-templates.md`。

---

## 一、工作流程（四阶段，不可跳过）

### 阶段一：对话式收集信息

向用户逐步确认以下信息。不全时追问，不假设。

```
1. 产品定位：解决什么问题？给谁用？
2. 验证假设：MVP 要验证什么？怎么算成功？（量化标准）
3. 功能边界：Must Have / Explicitly Excluded / Nice to Have
4. 技术选型：见下方选型建议表。如果用户不确定，推荐默认组合。
5. AI 模型：需要接入哪些 provider？（图片/视频/文本/语音）
6. 视觉方向：参考竞品 / 风格关键词 / 暗色 or 亮色
```

**技术选型推荐（用户不确定时用这套）**：

| 层 | 推荐 | 理由 |
|----|------|------|
| 语言 | TypeScript | AI编程工具最擅长、类型系统最强 |
| 前端 | Next.js + React | 全栈一体、SSR/ISR、API Routes |
| CSS | Tailwind CSS | 与AI编程配合度最高、utility-first |
| 后端 | Next.js API Routes 或 Express | 前者零配置、后者更灵活 |
| 数据库 | PostgreSQL (Supabase) | 托管+Auth+Realtime一体化 |
| 文件存储 | Cloudflare R2 | 零出口费、S3兼容 |
| 部署 | Vercel | 自动部署、CDN、Serverless |
| 状态管理 | Zustand | 轻量、与AI编程配合好 |

收到回答后输出「项目简报」让用户确认，确认后进入阶段二。

### 阶段二：生成 8 个架构文档

⚠️ **必须在同一执行流程中生成全部 8 个文档。** 这是它们一致性最高的时刻。

生成顺序（后面依赖前面，严格遵守）：

| 序号 | 文档 | 存放路径 | 模板来源 |
|:---:|------|---------|---------|
| 1 | 验证目标 | `docs/VALIDATION_TARGET.md` | `references/templates/validation-target.md` |
| 2 | 数据模型 | `docs/DATA_MODEL.md` | `references/templates/doc-templates.md` §DM |
| 3 | API契约 | `docs/API_CONTRACT.md` | `references/templates/doc-templates.md` §AC |
| 4 | 设计系统 | `docs/DESIGN_SYSTEM.md` | `references/templates/doc-templates.md` §DS |
| 5 | Agent规范 | `AGENTS.md`（项目根） | `references/templates/doc-templates.md` §AG |
| 6 | 开发约束 | `CLAUDE.md`（项目根） | `references/templates/doc-templates.md` §CL |
| 7 | 任务看板 | `docs/TASK_BOARD.md` | `references/templates/doc-templates.md` §TB |
| 8 | 部署清单 | `docs/DEPLOY_CHECKLIST.md` | `references/templates/doc-templates.md` §DC |

**耦合约束（生成时必须遵循）**：
- 每个实体标注来源功能（→ VT）
- API 请求/响应体类型引用 DATA_MODEL 定义
- AGENTS 每个工具在 API_CONTRACT 映射表中有对应
- CLAUDE 严禁清单每条标注来源 + 强制执行机制
- provider 枚举在 DM、AC、AGENTS 三处完全一致
- DEPLOY_CHECKLIST 覆盖 VT 所有 Must Have 的回归路径

### 阶段三：生成约束代码骨架

读取 `references/templates/constraint-code.md`，基于项目信息填充模板变量。

**生成的代码文件**（路径根据技术选型调整）：

类型约束层：
- `src/types/provider.ts` — Provider单一来源 + Record强制补全
- `src/types/task-status.ts` — 状态机 + validateTransition + assertNever
- `src/types/url.ts` — Branded URL 类型
- `src/types/api-response.ts` — 错误码枚举 + 响应类型

运行时守卫层：
- `src/lib/api-response.ts` — apiSuccess / apiError 构造函数
- `src/lib/data-service.ts` — 数据层唯一入口 + URL写入拦截
- `src/middleware/confirm-guard.ts` — 确认拦截中间件

检查脚本层：
- `scripts/lint-constraints.js` — 自定义约束扫描
- `scripts/validate-consistency.py` — 文档一致性校验

配置层：
- `.eslintrc.js` 补充规则（no-restricted-imports）

**每个代码文件头部写 10-20 行约束注释**，只含与该文件直接相关的约束。
不让 AI 去查 CLAUDE.md 大海捞针，而是在编辑时直接看到约束。

### 阶段四：校验与交付

1. 运行 `python scripts/validate-consistency.py .` 确认文档一致
2. 运行 `node scripts/lint-constraints.js src/` 确认代码骨架合规
3. 列出全部生成的文件清单
4. 输出下一步建议：
   - "用 harvest-dev Skill 拆解首批任务"
   - "开始实现 TASK_BOARD 中的第一个任务"
   - "部署前用 deploy-gate Skill 做门禁检查"

---

## 二、环境适配说明

本 Skill 在以下环境中均可工作：

| 环境 | 文件操作方式 | 脚本运行方式 |
|------|-----------|-----------|
| Claude Code | 直接 create_file / bash | bash 直接运行 |
| Claude.ai（本环境） | create_file + bash_tool | bash_tool 运行 |
| Cursor / Windsurf | 生成文件内容，IDE 保存 | 终端运行 |
| Codex / Antigravity | 按 CLAUDE.md 约束生成 | 终端运行 |

在非终端环境中，脚本内容会作为文件输出，用户需自行运行。

---

## 三、约束代码的设计原则

1. **能用类型系统约束的，不用文档约束** — Branded Type, exhaustive switch, Record
2. **能用运行时拦截的，不用人工检查** — data-service 守卫, confirm-guard
3. **能用 lint 规则检查的，不用 AI 自觉** — ESLint, 自定义脚本
4. **约束就近内联** — 文件头部注释，不查大文件

---

## Related Skills

| Skill | 何时使用 |
|-------|---------|
| **saas-quickstart** | 不需要完整骨架，只想快速基于 starter kit 上线 |
| **deploy-gate** | 部署前门禁检查（使用本 skill 生成的约束脚本做关2检查） |
| **nextjs-fullstack** | Next.js App Router 全栈开发模式参考 |
| **mvp-billing-system** | 在骨架基础上接入计费系统 |
