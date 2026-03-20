---
name: readme-standard
description: 标准化 GitHub README 写作skill。当用户要写README、更新README、创建项目文档、优化项目介绍时触发。生成居中徽章头部、emoji章节标题、编号优势列表、架构图、详细表格、FAQ等完整结构。支持双语（EN+CN）。关键词：README、项目介绍、文档、write readme、update readme。
---

# README Standard — GitHub 项目 README 标准化写作指南

> 生成专业、用户友好、视觉层次分明的 GitHub README。

---

## 一、触发条件

当用户提到以下关键词时激活：
- "写 README"、"更新 README"、"优化 README"
- "项目介绍"、"项目文档"
- "write readme"、"update readme"、"project documentation"

---

## 二、工作流程（三阶段）

### 阶段一：信息收集

从项目上下文中提取以下信息（优先自动推断，不足时追问）：

```
1. 项目名称与一句话定位
2. GitHub 仓库地址（用于生成 badges）
3. 核心功能 / 优势（3-6 条）
4. 技术栈
5. 安装方式（一键安装 / 分步安装）
6. 架构概览（模块关系）
7. 是否需要双语版本（EN + CN）
8. 许可证类型
```

### 阶段二：生成 README

严格按照下方「标准结构」生成。**所有章节按顺序输出，不可跳过。**

### 阶段三：校验

- [ ] 所有 badge URL 中的 `{owner}/{repo}` 已替换为真实值
- [ ] 架构图与实际模块一致
- [ ] 安装命令可直接复制执行
- [ ] 双语版本（如需要）结构对齐、内容一致
- [ ] 无占位符残留（搜索 `TODO`、`xxx`、`your-`）

---

## 三、标准结构（9 个章节，严格顺序）

### 章节 0：居中头部（Header）

```markdown
<div align="center">

# 项目名称

**一句话定位描述**

[![GitHub Stars](https://img.shields.io/github/stars/{owner}/{repo}?style=flat-square)](https://github.com/{owner}/{repo}/stargazers)
[![GitHub Forks](https://img.shields.io/github/forks/{owner}/{repo}?style=flat-square)](https://github.com/{owner}/{repo}/network)
[![GitHub Issues](https://img.shields.io/github/issues/{owner}/{repo}?style=flat-square)](https://github.com/{owner}/{repo}/issues)
[![GitHub Pull Requests](https://img.shields.io/github/issues-pr/{owner}/{repo}?style=flat-square)](https://github.com/{owner}/{repo}/pulls)

[![GitHub License](https://img.shields.io/github/license/{owner}/{repo}?style=flat-square)](https://github.com/{owner}/{repo}/blob/main/LICENSE)
[![Version](https://img.shields.io/badge/version-v1.0.0-green.svg?style=flat-square)](https://github.com/{owner}/{repo})
[![自定义徽章](https://img.shields.io/badge/{label}-{value}-{color}.svg?style=flat-square)](URL)

[English](./README.md) | [中文文档](./README_CN.md)

</div>
```

**规则**：
- 使用 `<div align="center">` 居中
- 徽章分两行：第一行社区指标（stars/forks/issues/PRs），第二行项目元数据（license/version/自定义）
- 统一 `style=flat-square`
- 语言切换链接放最后
- 自定义徽章用于突出项目特色（如技能数量、代码行数、覆盖率等）

---

### 章节 1：提示框（可选）

```markdown
> [!NOTE]
> 一段重要说明，解释项目核心概念或前置知识。
```

**使用场景**：
- 项目有特殊概念需要先解释（如 "什么是 Claude Code Skill"）
- 有重要公告或版本迁移说明
- 类型：`[!NOTE]`（信息）、`[!IMPORTANT]`（重要）、`[!WARNING]`（警告）

---

### 章节 2：⚡ 项目概述（Overview）

```markdown
## ⚡ Overview

**{项目名}** 是 ...（2-3 句定位描述）

相比同类产品/方案，我们拥有 🚀 **N 大优势**：

1. **优势标题一**：详细描述（1-2 句，具体而非泛泛）。

2. **优势标题二**：详细描述。

3. **优势标题三**：详细描述。
```

**规则**：
- 优势数量 3-6 条，每条「**加粗标题** + 冒号 + 详细描述」
- 描述必须具体，带数字或对比（"1,300+ lines"、"saves you weeks"），避免空洞形容词
- 最后可加一句升华定位（"始于 X，而不止于 X"）
- 引用块 `>` 用于补充说明或技术栈适用范围

---

### 章节 3：📦 快速安装（Quick Start）

```markdown
## 📦 Quick Start

### Option 1: 推荐方式

\```bash
# 一键安装命令
\```

### Option 2: 按需安装

\```bash
# 分步安装命令
\```

### Verify Installation

\```bash
# 验证命令
\```
```

**规则**：
- 推荐方式放第一个，标注 "(Recommended)"
- 每个代码块前有 `#` 注释说明
- 必须包含验证步骤（用户怎么确认安装成功）
- 命令可直接复制执行，无需手动替换变量

---

### 章节 4：🏗️ 架构（Architecture）

```markdown
## 🏗️ Architecture

### 架构图

\```
（ASCII / Box-drawing 架构图）
\```

### 模块详情表

| # | 模块 | 描述 |
|:-:|------|------|
| 1 | **模块名** | 功能描述 |
```

**架构图规则**：
- 使用 Box-drawing 字符（`┌ ┐ └ ┘ │ ─ ├ ┤ ┬ ┴ ┼ ▼ ▶`）或简单 ASCII
- 标注每个模块的用户触发语句（如 "Start a new project"）
- 包含模块间的数据流方向
- 宽度不超过 70 字符（移动端友好）

**模块表规则**：
- 列：`#` | 名称 | 量化指标（行数/文件数）| 触发条件 | 功能描述
- 序号列居中 `:-:`
- 名称加粗
- 数字右对齐 `---:`

---

### 章节 5：🔍 详细介绍（Deep Dive）

```markdown
## 🔍 Deep Dive

### 1. 模块名称

> **一句话定位**

| 阶段/步骤 | 操作 | 产出 |
|:---------:|------|------|
| 1 | ... | ... |

---

### 2. 下一个模块

> **一句话定位**

（内容）

---
```

**规则**：
- 每个模块/功能一个 `###` 子节
- 以 `>` 引用块开头，一句话定位
- 表格呈现结构化信息（阶段、步骤、参数配置）
- 模块间用 `---` 分隔
- 关键提示用 `> **Note**:` 格式

---

### 章节 6：⚙️ 技术栈（Tech Stack）

```markdown
## ⚙️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | TypeScript |
| Frontend | Next.js 14/15+ |
| ... | ... |
```

**规则**：
- 两列表格：层 | 技术
- 如果技术栈可替换，加一句说明："Many patterns are stack-agnostic."

---

### 章节 7：🔧 自定义（Customization）

```markdown
## 🔧 Customization

1. **操作一** — 说明
2. **操作二** — 说明

### File Structure

\```
项目目录/
├── 文件夹/
│   ├── 文件                    # 注释说明
│   └── 文件                    # 注释说明
└── 文件                        # 注释说明
\```
```

**规则**：
- 自定义步骤用编号列表
- 文件结构树使用 `├── └── │` 符号
- 每个文件/目录带 `# 注释说明`，右对齐（空格补齐到统一列位）

---

### 章节 8：❓ FAQ

```markdown
## ❓ FAQ

**Q: 问题一？**
> 回答，用引用块格式。

**Q: 问题二？**
> 回答。可包含代码块：
> \```bash
> 命令
> \```
```

**规则**：
- 问题加粗，以 `Q:` 开头
- 回答用 `>` 引用块
- 预设 4-6 个常见问题
- 必须包含：是否必须全部安装、是否有冲突、如何更新、如何贡献

---

### 章节 9：📄 许可证（License）

```markdown
## 📄 License

[MIT](./LICENSE) — Use freely, modify freely, no attribution required.
```

**规则**：
- 一行即可，链接到 LICENSE 文件
- 加一句人话说明许可含义

---

## 四、双语版本规范

当需要生成双语版本时：

| 项目 | 规则 |
|------|------|
| 文件命名 | `README.md`（主语言）+ `README_CN.md`（中文）或 `README-EN.md`（英文） |
| 头部切换 | 两个文件头部都放 `[English](./README.md) \| [中文文档](./README_CN.md)` |
| 结构对齐 | 两个版本章节数量、顺序、表格行数完全一致 |
| 徽章共用 | badges 两个版本完全相同（shields.io 是英文的，不翻译） |
| 技术术语 | 代码、命令、技术名词不翻译（Next.js, Supabase, Stripe, TypeScript） |
| 中文排版 | 中英文之间加空格（如「使用 Next.js 构建」而非「使用Next.js构建」） |

---

## 五、Emoji 章节标题速查

| Emoji | 含义 | 用于 |
|:-----:|------|------|
| ⚡ | 闪电 / 快速 | 项目概述（Overview） |
| 📦 | 包裹 | 安装（Quick Start / Installation） |
| 🏗️ | 建筑 | 架构（Architecture） |
| 🔍 | 放大镜 | 详细介绍（Deep Dive / Details） |
| ⚙️ | 齿轮 | 技术栈 / 配置（Tech Stack / Configuration） |
| 🔧 | 扳手 | 自定义 / 开发（Customization / Development） |
| 🚀 | 火箭 | 部署 / 优势（Deploy / Advantages） |
| ❓ | 问号 | 常见问题（FAQ） |
| 📄 | 文件 | 许可证（License） |
| 📊 | 图表 | 数据 / 指标（Metrics / Stats） |
| 🛡️ | 盾牌 | 安全（Security） |
| 🧪 | 试管 | 测试（Testing） |
| 📖 | 书 | 文档 / 指南（Docs / Guide） |

---

## 六、Shields.io 徽章模板库

### 社区指标（第一行）

```markdown
[![GitHub Stars](https://img.shields.io/github/stars/{owner}/{repo}?style=flat-square)](https://github.com/{owner}/{repo}/stargazers)
[![GitHub Forks](https://img.shields.io/github/forks/{owner}/{repo}?style=flat-square)](https://github.com/{owner}/{repo}/network)
[![GitHub Issues](https://img.shields.io/github/issues/{owner}/{repo}?style=flat-square)](https://github.com/{owner}/{repo}/issues)
[![GitHub Pull Requests](https://img.shields.io/github/issues-pr/{owner}/{repo}?style=flat-square)](https://github.com/{owner}/{repo}/pulls)
```

### 项目元数据（第二行）

```markdown
[![GitHub License](https://img.shields.io/github/license/{owner}/{repo}?style=flat-square)](https://github.com/{owner}/{repo}/blob/main/LICENSE)
[![Version](https://img.shields.io/badge/version-v1.0.0-green.svg?style=flat-square)](https://github.com/{owner}/{repo})
```

### 常用自定义徽章

```markdown
<!-- 技术栈 -->
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-3178C6?style=flat-square&logo=typescript&logoColor=white)]()
[![Next.js](https://img.shields.io/badge/Next.js-15-000000?style=flat-square&logo=next.js&logoColor=white)]()
[![React](https://img.shields.io/badge/React-19-61DAFB?style=flat-square&logo=react&logoColor=black)]()
[![Node.js](https://img.shields.io/badge/Node.js-20-339933?style=flat-square&logo=node.js&logoColor=white)]()
[![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&logo=python&logoColor=white)]()

<!-- 工具 -->
[![Docker](https://img.shields.io/badge/Docker-Build-2496ED?style=flat-square&logo=docker&logoColor=white)]()
[![Vercel](https://img.shields.io/badge/Vercel-Deploy-000000?style=flat-square&logo=vercel&logoColor=white)]()
[![Supabase](https://img.shields.io/badge/Supabase-Database-3FCF8E?style=flat-square&logo=supabase&logoColor=white)]()
[![Stripe](https://img.shields.io/badge/Stripe-Payments-635BFF?style=flat-square&logo=stripe&logoColor=white)]()

<!-- 状态 -->
[![CI](https://img.shields.io/github/actions/workflow/status/{owner}/{repo}/ci.yml?style=flat-square&label=CI)]()
[![Coverage](https://img.shields.io/badge/coverage-85%25-brightgreen?style=flat-square)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)]()

<!-- 量化 -->
[![Lines](https://img.shields.io/badge/lines-5%2C366-orange.svg?style=flat-square)]()
[![Skills](https://img.shields.io/badge/skills-9-blue.svg?style=flat-square)]()
```

---

## 七、反面模式（避免）

| 反面模式 | 正确做法 |
|---------|---------|
| 徽章过多（>10 个），堆砌无意义 badge | 只放与用户决策相关的 badge（stars、license、version） |
| 优势描述空洞（"高性能"、"易使用"） | 带具体数字或对比（"1,300+ lines"、"saves weeks"） |
| 安装步骤需要手动替换变量 | 提供可直接复制的命令，变量在前文说明 |
| 没有验证步骤 | 每次安装后告诉用户如何确认成功 |
| 架构图超宽（>80 列） | 控制在 70 列以内，移动端可读 |
| FAQ 过少（<3 条） | 至少 4 条，覆盖安装、兼容、更新、贡献 |
| 文件结构树无注释 | 每个文件/目录带 `# 说明` |
| 中英文之间无空格 | 中英文之间加空格（如「使用 Next.js」） |
| 双语版本结构不对齐 | 两个文件章节数、表格行数完全一致 |

---

## Related Skills

| Skill | 何时使用 |
|-------|---------|
| **project-scaffold** | 生成项目骨架时，其中的 DEPLOY_CHECKLIST 可作为 README 部署章节的来源 |
| **deploy-gate** | 部署前检查，确认 README 中的安装命令、环境变量清单与代码一致 |
| **doc-audit** | 文档一致性审计，检测 README 与代码实际状态是否匹配 |
