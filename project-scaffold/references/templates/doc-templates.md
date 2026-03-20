# 架构文档模板合集

> DATA_MODEL / API_CONTRACT / DESIGN_SYSTEM / AGENTS / CLAUDE / DEPLOY_CHECKLIST
> `{{ }}` 根据项目信息填充。每个模板标注了它的上游依赖和下游消费者。

---

## §DM — DATA_MODEL.md

上游：VALIDATION_TARGET §2.1
下游：API_CONTRACT（类型引用）、AGENTS（工具参数）、CLAUDE（数据操作规范）
代码对应：`src/types/provider.ts` + `src/types/task-status.ts` + `src/types/url.ts`

```markdown
# {{ 项目名 }} 数据模型

## 来源声明
实体来源于 VALIDATION_TARGET.md §2.1。每个实体标注来源功能。

## 1. 实体关系图
{{ Mermaid 或文字 }}

## 2. 核心实体

### 2.1 {{ 业务实体 }}
用途：{{ }} | 来源功能：VT §2.1 {{ }}
| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|:---:|--------|------|
代码：→ `src/types/{{ entity }}.ts`

### 2.2 UnifiedTask（统一任务表）
用途：所有AI模型的生成任务，provider字段区分 | 来源功能：VT中涉及AI生成的全部功能
| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|:---:|--------|------|
| id | uuid | ✅ | auto | 主键 |
| provider | text | ✅ | | → `src/types/provider.ts` PROVIDERS |
| status | text | ✅ | 'queued' | → `src/types/task-status.ts` |
| input_params | jsonb | ✅ | | 各provider特有参数 |
| output_url | text | | | 可临时 |
| persistent_url | text | | | 必须持久化 → `src/types/url.ts` PersistentUrl |
| credits_estimated | int | ✅ | | ≥0 |
| user_confirmed | boolean | ✅ | false | 确认标记 |
| retry_count | int | ✅ | 0 | ≤3 |
| error_message | text | | | |
| metadata | jsonb | | {} | |
| created_at | timestamptz | ✅ | now() | |
| updated_at | timestamptz | ✅ | now() | |

## 3. 状态机
完整定义在 `src/types/task-status.ts`（代码即文档）。
状态转移图：queued→processing→completed/failed; 任意→cancelled。

## 4. 索引策略
| 表 | 索引 | 用途 |
|----|------|------|

## 5. 临时URL持久化规则
完整实现在 `src/types/url.ts` + `src/lib/data-service.ts`（代码即文档）。
规则：output_url 允许临时；persistent_url 只接受 PersistentUrl。
```

---

## §AC — API_CONTRACT.md

上游：VT（功能来源）、DM（类型来源）
下游：AGENTS（工具映射）、CLAUDE（开发规范）、confirm-guard.ts（确认端点列表）

```markdown
# {{ 项目名 }} API 契约

## 来源声明
- 端点来源：VT §2.1 | 类型来源：DM §2
- 响应格式代码：`src/types/api-response.ts` + `src/lib/api-response.ts`

## 1. 错误码表
→ 代码定义在 `src/types/api-response.ts` ERROR_CODES
| 错误码 | HTTP | 含义 |
|--------|:---:|------|
| UNAUTHORIZED | 401 | 未认证 |
| FORBIDDEN | 403 | 无权限 |
| NOT_FOUND | 404 | 不存在 |
| INSUFFICIENT_CREDITS | 402 | 积分不足 |
| PROVIDER_ERROR | 502 | AI服务错误 |
| RATE_LIMITED | 429 | 频率限制 |
| TIMEOUT | 504 | 超时 |
| VALIDATION_ERROR | 422 | 参数错误 |
| CONFIRMATION_REQUIRED | 428 | 需确认 |

## 2. 端点清单

### {{ METHOD }} {{ PATH }}
用途：{{ }} | 来源功能：VT §2.1 {{ }}
认证：✅/❌ | 积分：✅/❌ | 确认：✅/❌
超时：{{ N }}ms（显式，不用默认值）
请求体：→ DM §{{ }} 类型
前置检查：认证→校验→积分→确认→业务

{{ 更多端点 }}

## 3. 端点-工具映射表
| API端点 | Agent工具名 | AGENTS章节 | 确认 |
|---------|-----------|-----------|:---:|
→ 此表与 AGENTS §3 双向一致
→ 此表与 `src/middleware/confirm-guard.ts` CONFIRMATION_REQUIRED 一致
```

---

## §DS — DESIGN_SYSTEM.md

上游：VT §3 用户画像
下游：CLAUDE（UI规范）、lint-constraints.js（检查规则）

```markdown
# {{ 项目名 }} 设计系统

## 来源声明
审美方向来源于 VT §3。

## 1. 设计哲学（≤100字）
{{ }}

## 2. 色彩体系
:root { {{ CSS变量 }} }
→ lint检查：`scripts/lint-constraints.js` R1 禁止硬编码颜色

## 3. 字体
| 用途 | 字体 | 字重 | 大小 |
禁用字体：{{ }}

## 4. 间距
4px网格：4/8/12/16/24/32/48/64

## 5. 组件规范
{{ 核心组件样式 }}

## 6. z-index管理
| 层级 | 范围 | 用途 |
|------|------|------|
| 基础 | 0-9 | 页面主体 |
| 浮层 | 10-19 | 侧边栏 |
| 弹窗 | 20-29 | 模态框 |
| 通知 | 30-39 | Toast |
| 顶层 | 40-49 | 遮罩 |
→ lint检查：`scripts/lint-constraints.js` R3

## 7. DO / DON'T
### ✅ DO
{{ }}
### ❌ DON'T
{{ }} → 每条对应 lint-constraints.js 中的检查规则
```

---

## §AG — AGENTS.md

上游：AC（端点映射）、DM（类型）、DS（DO/DON'T）、VT（排除项）
下游：CLAUDE（工具实现规范）、confirm-guard.ts（确认列表）

```markdown
# {{ 项目名 }} AI Agent 行为规范

## 来源声明
- 工具列表 ← AC §3 映射表（双向一致）
- 数据类型 ← DM + `src/types/`
- 确认列表 ← AC "确认:✅" 端点 = `confirm-guard.ts` CONFIRMATION_REQUIRED

## 1. 能力范围
可以：{{ }}
不可以：{{ 来自 VT §2.2 排除项 }}

## 2. 确认协议
必须确认的操作：{{ 与 AC 确认端点一致 }}
流程：指令→解析→展示计划（操作/参数/预估积分）→确认→执行
抽风检测：准备不调用工具就输出结果时，立即停止。

## 3. 工具定义
### 3.1 {{ tool_name }}
端点：AC §2 {{ }}
确认：✅/❌
参数：→ `src/types/` 中的类型

## 4. 状态约束
→ `src/types/task-status.ts`
只有 completed + persistent_url 存在时才说"完成"

## 5. 严禁行为
{{ 每条标注来源 }}
```

---

## §CL — CLAUDE.md

上游：消费以上全部
下游：TASK_BOARD（约束来源）、每个代码文件头部注释（约束分发）

```markdown
# {{ 项目名 }} 开发规范

## 文档依赖
修改本文档时检查是否与上游矛盾。

## 1. 项目结构
{{ 目录树 }}

## 2. 命名规范
| 类型 | 约定 | 示例 |

## 3. 数据操作
→ 唯一入口：`src/lib/data-service.ts`（ESLint强制）
→ URL持久化：`src/types/url.ts`（运行时强制）

## 4. API开发
→ 响应格式：`src/lib/api-response.ts`（编译时强制）
→ 超时显式声明（lint检查）
→ 中间件顺序：认证→参数→积分→确认→业务

## 5. UI开发
→ 颜色用CSS变量（lint检查）
→ 禁止厚重阴影（lint检查）
→ z-index在范围内（lint检查）

## 6. 严禁清单
| ❌ 禁止行为 | 来源 | 强制机制 |
|------------|------|---------|
| 存储临时URL到持久化字段 | DM §3.2 | 编译时(BrandedType) + 运行时(data-service) |
| 绕过data-service操作数据库 | CLAUDE §3 | ESLint规则 + lint脚本 |
| 硬编码颜色值 | DS §2.2 | lint脚本 |
| 使用厚重阴影 | DS §7 | lint脚本 |
| z-index超范围 | DS §6 | lint脚本 |
| 非法状态跳转 | DM §3.1 | 运行时(validateTransition) |
| 遗漏新provider配置 | DM §2.2 | 编译时(Record) |
| 遗漏新状态处理 | DM §3.1 | 编译时(assertNever) |
| 未确认就执行消耗操作 | AG §2.1 | 运行时(confirm-guard) |
| API用默认超时 | AC §2 | lint脚本 |

## 7. 开发流程
实施顺序：类型→Service→API→Hook→UI
3次失败后停止→记录→替代方案→质疑假设

## 8. Git规范
main只接受合并 | feat:/fix:/refactor:/docs:
```

---

## §DC — DEPLOY_CHECKLIST.md

上游：消费全部文档 + 约束脚本

```markdown
# {{ 项目名 }} 部署清单

## 第一关：编译
- [ ] npm run build 零错误
- [ ] tsc --noEmit 通过

## 第二关：约束脚本
- [ ] node scripts/lint-constraints.js 通过
- [ ] python scripts/validate-consistency.py 通过

## 第三关：功能回归
{{ 来自 VT §2.1 Must Have，每个功能一条路径 }}

## 第四关：Agent行为
- [ ] 消耗操作弹出确认
- [ ] 工具列表与 AC §3 一致

## 第五关：边界
- [ ] 刷新后数据完整
- [ ] 积分不足有提示
- [ ] 取消任务不报错

## 第六关：文档同步
- [ ] CHANGELOG 已更新
- [ ] 版本号已更新
```
