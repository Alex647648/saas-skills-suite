---
name: deploy-gate
description: 部署前门禁检查。在部署到生产环境前依次执行六关检查：编译、约束脚本扫描、环境变量、核心功能回归、Git分支、最终确认。任何一关失败即阻断部署。当用户提到"部署"、"发布"、"上线"、"deploy"、"发版"、"推到生产"、"能不能上线"时触发。也可随时运行做项目健康检查。
---

# Deploy Gate — 部署门禁

> 部署前最后一道防线。六关全部通过才放行。

---

## 六关检查

### 关1：编译

```bash
npm run build
npx tsc --noEmit
npx eslint src/ --quiet
```

任一失败 → 阻断，报告错误。

### 关2：约束脚本

```bash
# 如果脚本存在则运行，不存在则跳过并提示
[ -f scripts/lint-constraints.js ] && node scripts/lint-constraints.js src/ || echo "SKIP: lint-constraints.js 不存在（可通过 project-scaffold skill 生成）"
[ -f scripts/validate-consistency.py ] && python scripts/validate-consistency.py . || echo "SKIP: validate-consistency.py 不存在（可通过 project-scaffold skill 生成）"
```

lint-constraints 检查：硬编码颜色、厚重阴影、z-index越界、缺超时、直接引用DB。
validate-consistency 检查：provider一致、端点-工具映射、严禁清单来源、文档完整。

> **容错策略**：这两个脚本由 **project-scaffold** skill 在阶段三生成。
> 如果项目未使用 project-scaffold 初始化，脚本可能不存在 — 此时跳过关2，
> 并在最终报告中标注"关2 跳过（无约束脚本）"。

任一失败 → 阻断，报告具体违规。脚本不存在 → 跳过并标注。

### 关3：环境检查

- 读取 git diff，检查新增的 `process.env.` 引用
- 如有，检查 `.env.example` 中是否有对应条目
- 数据库 Schema 变更是否有迁移文件
- 迁移是否可逆（DOWN Migration）

### 关4：功能回归

读取 `docs/DEPLOY_CHECKLIST.md`（如存在），展示回归清单。
如不存在，从 `docs/VALIDATION_TARGET.md` §2.1 Must Have 生成。

展示清单让用户逐项确认。

### 关5：Git 检查

```bash
git status --porcelain     # 应为空
git branch --show-current  # 应为 main 或合并分支
```

检查 CHANGELOG 是否已更新。

### 关6：最终汇总

```markdown
## 部署门禁报告 — {{ 时间 }}

### 自动检查
- ✅/❌ 关1 编译：{{ }}
- ✅/❌ 关2 约束脚本：{{ }}
- ✅/❌ 关3 环境：{{ }}
- ✅/❌ 关5 Git：{{ }}

### 需人工确认
- [ ] 关4 功能回归（{{ N }}项）

### 结论
{{ ✅ 可部署 / ❌ 阻断：原因 }}
```

---

## 回滚提醒

每次部署前确认：
1. 平台支持一键回滚？
2. Schema变更有DOWN Migration？
3. 环境变量变更已记录在 .env.example？

---

## Related Skills

| Skill | 何时使用 |
|-------|---------|
| **project-scaffold** | 生成关2所需的约束脚本（lint-constraints.js + validate-consistency.py） |
| **mvp-billing-system** | 计费系统专用的部署检查清单（Stripe webhook、Edge Function 环境变量） |
| **supabase-gemini-deploy** | Edge Function 部署后的排查手册（401/500/CORS） |
| **nextjs-fullstack** | Next.js + Vercel 部署流程 |
