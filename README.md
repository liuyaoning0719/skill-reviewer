# Workflow Skill Reviewer v1.4 — 使用指南

> 一个用于审查 Agent Skill 设计质量的工具。在 Skill 发布前检查结构设计、触发精准度、工具权限和安全隐患。

---

## 这是什么

`workflow-skill-reviewer` 审查的是 **Skill 文件本身**（SKILL.md），而不是 Agent 的执行行为。它回答的问题是：**"这个 Skill 的设计是否规范？"**

与之配套的 `execution-audit` 回答的是：**"Agent 执行这个 Skill 时有没有偷懒、瞎编、过度工程化？"**

| | workflow-skill-reviewer | execution-audit |
|:---|:---|:---|
| 审查对象 | SKILL.md 文件（Skill 设计） | Agent 执行日志（运行时行为） |
| 审查时机 | 创建 / 更新 / 发布前 | 执行时或执行后 |
| 检测重点 | 结构、触发词、工具权限、渐进披露 | 偷懒、跳步、幻觉、过度工程 |

---

## 核心能力

### 7 维度审查框架

每个 Skill 从这 7 个维度被评估：

| 维度 | 优先级 | 核心问题 |
|:---|:---:|:---|
| **T1 — 触发精准度** | Critical | description 是否只在正确的场景被激活？ |
| **S1 — 结构保真度** | Critical | 是否有编号阶段、入口标准、出口标准？ |
| **S2 — 指令可执行性** | High | 步骤是否足够确定，防止 Agent 幻觉？ |
| **S3 — 上下文效率** | High | 是否只加载需要的内容？ |
| **C1 — 漂移抗性** | High | 边缘输入和失败状态下是否保持形状？ |
| **S4 — 工具最小化** | Medium | 工具是否精确匹配工作，无过度授权？ |
| **S5 — 范围一致性** | Medium | 是否聚焦单一工作流，无范围蔓延？ |

### 6 阶段审查流程

```
Phase 1: 结构验证 → Phase 2: 触发分析 → Phase 3: 模式识别
→ Phase 4: 内容质量 → Phase 5: 工具审查 → Phase 6: 评分报告
```

每个阶段都有明确的**入口标准**、**动作清单**和**出口标准**。

### 27 项反模式（AP-1 ~ AP-27）

部分典型反模式：

| ID | 反模式 | 严重程度 | 一句话修复 |
|:---|:---|:---:|:---|
| AP-1 | 缺少 When to Use / When NOT to Use | Critical | 添加两个区块，写明替代方案 |
| AP-2 | SKILL.md 过于臃肿（>500行） | Critical | 拆分到 `references/` 和 `scripts/` |
| AP-6 | 阶段无编号 | Critical | 每个阶段加编号、入口/出口标准 |
| AP-10 | description 总结工作流内容 | Critical | description 只写触发条件 |
| AP-11 | 该用专用工具时用了 Bash | High | 文件操作用 Glob/Grep/Read，不用 Bash |
| AP-21 | requires 指向不存在的 Skill | High | 删除幽灵依赖或创建对应 Skill |
| AP-24 | Flow Skill 缺少流程图 | Critical | 添加 Mermaid/D2 图，标 BEGIN/END |
| AP-27 | 硬编码凭证 | Critical | 用 `{{API_KEY}}` 占位符 |

> 完整清单见 Skill 目录下的 `references/anti-patterns.md`

---

## 快速开始

### 1. 审查单个 Skill

```bash
# 基础审查
python skills/workflow-skill-reviewer/scripts/validate_skill.py /path/to/your-skill

# 带父目录（可验证跨 Skill 依赖）
python skills/workflow-skill-reviewer/scripts/validate_skill.py /path/to/your-skill --parent-dir /path/to/skills-suite
```

输出示例：
```
Validating skill: my-skill
==================================================
  PASS SKILL.md exists
  PASS name 'my-skill' is valid
  WARN SKILL.md is 420 lines (ideal: < 300)
  PASS Directory structure is flat
  ...
==================================================
All critical checks passed.
```

### 2. 审查整个 Skill 套件

```bash
python skills/workflow-skill-reviewer/scripts/validate_skill.py --suite /path/to/skills-suite
```

套件审查会执行：
- **Phase A**：逐个验证每个 Skill（20 项检查）
- **Phase B**：跨 Skill 关系分析（依赖解析、幽灵检测、层级映射、循环依赖、脚本重复）

### 3. AI 人工审查

将目标 Skill 的 SKILL.md 内容提供给执行 `workflow-skill-reviewer` 的 Agent，它会按 7 维度框架逐项审查并输出结构化报告。

---

## 20 项自动化检查清单

`validate_skill.py` 自动执行以下检查：

```
 1. SKILL.md 存在
 2. YAML frontmatter 合法
 3. name 字段合规（小写、连字符、≤64字符）
 4. description 存在（≤1024字符）
 5. 无多余 frontmatter 字段
 6. 行数 ≤500（理想 <300）
 7. 无禁止文档文件（CHANGELOG/INSTALLATION_GUIDE 等）
 8. 目录严格一层深（无嵌套）
 9. references/ 无循环引用链
10. SKILL.md 内文件引用均可解析
11. 无硬编码绝对路径
12. 包含 When to Use / When NOT to Use
13. Flow Skill 有 BEGIN/END 标记
14. requires 依赖的 Skill 存在
15. 跨 Skill 脚本无重复
16. 架构描述与目录结构一致
17. Flow Skill 有 Mermaid/D2 图
18. description 不泄露实现细节
19. 有韧性机制（timeout/retry/fallback）
20. 无硬编码凭证
```

---

## 5 种工作流模式

| 模式 | 适用场景 | 结构特征 |
|:---|:---|:---|
| **Routing** | 多意图入口 | 路由表映射意图到工作流文件 |
| **Sequential Pipeline** | 步骤间有数据依赖 | 步骤串联，支持断点续传 |
| **Linear Progression** | 单一路径任务 | 编号阶段，每阶段有入口/出口标准 |
| **Safety Gate** | 破坏性操作前确认 | 两道确认门 |
| **Task-Driven** | 多任务并行/依赖管理 | 任务追踪，显式依赖声明 |

---

## 设计理念

1. **Description 即触发器** — Agent 仅凭 YAML frontmatter 的 `name` + `description` 决定是否激活 Skill。description 必须包含触发词、适用场景、排除场景。

2. **模式优于散文** — 无编号的散文指令会导致执行顺序不可靠。每个工作流必须有编号阶段、入口标准、出口标准。

3. **渐进式披露** — SKILL.md 是导航脑，不是百科全书。控制在 500 行以内，细节卸载到 `references/` 和 `scripts/`。

4. **最小权限工具** — 只列出 Skill 实际使用的工具。能用 `ReadFile`/`WriteFile`/`Glob`/`Grep` 解决的，不要用 `Bash`。

5. **Gotchas 是经验** — 每个 Skill 都应该有 Gotchas 段落，记录团队特定的坑。通用建议（"API 需要鉴权"）是噪音；具体边界情况（"Token X 30 分钟后过期"）才是信号。

---

## 目录结构

```
skills/workflow-skill-reviewer/
├── SKILL.md                          # Skill 定义文件（给 AI 看）
├── scripts/
│   └── validate_skill.py             # 自动化验证脚本（20 项检查 + Suite Mode）
└── references/
    ├── anti-patterns.md              # 27 项反模式完整定义
    ├── detailed-audit-criteria.md    # 详细审查标准
    ├── evaluation-rubric.md          # 评分规则
    ├── workflow-patterns.md          # 5 种模式结构模板
    ├── agents-md-guide.md            # AGENTS.md 审查指南
    ├── review-checklist.md           # 作者自查清单
    └── audit-report-template.md      # 报告模板
```

---

## 版本历史

| 版本 | 日期 | 主要变更 |
|:---|:---|:---|
| v1.0 | 2026-05-04 | 初始版本：7 维度、20 反模式、5 模式、6 阶段流程 |
| v1.1 | 2026-05-04 | 增加 execution-audit 区分、Rationalizations Table、JSON 中间输出 |
| v1.2 | 2026-05-04 | 增加 Gotchas、自由度校准、三层架构约束 |
| v1.3 | 2026-05-05 | **关键修复**：恢复严格"一层深"规则；增加 `--suite` 套件模式；增加 AP-21/22/23 |
| v1.4 | 2026-05-05 | 增加 Flow Skill 验证（Mermaid/D2、BEGIN/END）；ASI-friendly description 检查；韧性机制检测；凭证隔离检查；增加 AP-24/25/26/27 |

