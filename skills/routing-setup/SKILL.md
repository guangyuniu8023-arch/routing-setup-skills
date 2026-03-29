---
name: routing-setup
description: 引导用户从零搭建具备 skill 路由能力的 Agent 骨架，收集需求、定义 skill 清单、按 9 项架构约束生成代码文件
type: guide
trigger: 用户想从零开始构建一个 plan-execute 路由 Agent
---

# /routing-setup

本 skill 分两阶段完成：**Phase A 需求收集** 和 **Phase B Agent 骨架搭建**。

> 架构约束的完整说明见 `docs/routing-setup.md`，本文件仅做流程引导。

---

## Phase A: 需求收集

### Step 1 — 项目基本信息

向用户询问以下信息，缺一不可：

1. **项目目标**：这个 Agent 要解决什么问题？面向谁？
2. **技术栈**：语言 & 框架（如 Python / FastAPI）、部署方式
3. **LLM 选择**：使用哪个模型做 Planner（如 GPT-4o、Claude 3.5 Sonnet）
4. **输入/输出类型**：用户会发什么（文本/图片/链接），Agent 应返回什么

收集完毕后，用一段摘要复述给用户确认。

### Step 2 — 定义 Skill 清单

使用 **Baseline Intent 表格** 格式，与用户共同填写每个 skill：

```
| Skill名称 | 一句话意图 | 类型 | 必须通过场景(3-5) | 不该走这里(2-3) | 模糊区域 | 竞争skill |
```

命名规则：
- **Skill 名称**：小写连字符，例如 `image-generation`、`video-editing`
- **资源类型**：使用中文，例如"图片"、"视频"、"文档"

每个 skill 至少填写 3 个"必须通过场景"和 2 个"不该走这里"场景，这些将作为后续路由测试的种子用例。

### Step 3 — 确认摘要

将 Step 1 的项目信息 + Step 2 的 Skill 清单合并为一份完整摘要，呈现给用户：

- 项目名称、技术栈、LLM
- Skill 数量及各 skill 的一句话意图
- 输入输出类型

**用户确认后才进入 Phase B。**

---

## Phase B: Agent 骨架搭建

### Step 4 — 生成目录结构

根据 Step 1-3 收集的信息，生成项目目录树。完整目录规范参考 `docs/routing-setup.md`。

目录树至少包含：
- `app/agent/` — Planner、Router 核心逻辑
- `skills/` — 每个 skill 一个子目录，含 `SKILL.md`
- `tests/` — 路由测试文件
- `app/models/` — 数据模型

### Step 5 — 用户确认目录

将目录树展示给用户，等待确认或调整后再继续。

### Step 6 — 按架构约束生成代码文件

逐文件生成代码，每个文件展示后等用户确认。代码必须遵循以下 **9 项架构约束**（详细说明见 `docs/routing-setup.md`）：

1. **SkillStore 三层加载** (A1) — scan/load/read_file 渐进加载，YAML parser 用正则避免冒号 bug
2. **Schema-only Noop Tools** (A5) — Pydantic schema + StructuredTool，区分 planner/exec tools，get_replan_tools() 过滤
3. **Router Node 三路分派** (A2) — 无媒体无文→require_upload，有媒体无文→clarify，有文→plan
4. **Plan Node 结构** (C2, H1) — create_plan → _tool_calls_to_plan，每 step 含 step_id + status
5. **Replan Node 独立函数** (H3, A3, A4, I1) — allowed_actions 过滤 + respond-before-generate 过滤 + has_executable 检查
6. **Execute Service 命令式循环** — while True 循环（非 StateGraph），respond hoisting，SSE streaming
7. **Resource 数据结构** (C1, H2) — 含 semantic_label/mode/is_latest_result，type 字段用中文
8. **SSE 事件类型** (I3) — message_delta/done、generation_progress、loading_pulse 等完整事件集
9. **Prompt 模板变量** (C3, H4, H5) — planner: {skill_catalog}/{resource_summary}，replan: {skill_content}/{allowed_actions_table}

### Step 7 — 首个 SKILL.md + 冒烟测试

从 Step 2 的 Skill 清单中选取第一个 skill，生成其 `SKILL.md` 文件。

然后创建冒烟测试，使用 4-tuple 格式：

```
(case_id, asset_desc, user_text, expected_skill)
```

冒烟测试必须覆盖：
- SkillStore 能扫描并加载该 skill
- Planner 能正确将测试用例路由到该 skill
- scan -> load 完整链路无报错

---

## 执行规则

- **每个 Step 必须等用户确认后再继续**，不跳过任何 Step
- Phase B 代码生成时，**逐文件展示并等确认**，不批量生成
- 冒烟测试必须覆盖 **SkillStore scan -> load 链路**
- 如用户中途修改需求，回到对应 Step 重新确认

---

## 完成后交接

骨架搭建完成后，提示用户下一步：

- 如果有多个 skill 待填充 -> 运行 **`/routing-skill-build`** 逐个完善剩余 skill 的 description 和实现
- 如果只有一个 skill -> 可选安装 **routing-quality-skills** 插件并运行 `/routing-layer1` 开始 TDD 路由质量调优
