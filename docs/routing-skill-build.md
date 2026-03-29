# /routing-skill-build — 逐 Skill 构建与路由质量衔接

## 概述

本技能覆盖 Skill 工程的 **Phase C**（逐 Skill 构建循环）和 **Phase D**（闭合与衔接）。Phase C 完成每个 skill 从定义到可测的完整链路；Phase D 自动生成路由质量所需的配置和测试脚手架，为后续路由质量调优做好准备。

本文档与平台无关——适用于任何使用 planner + skill 路由架构的 agent 项目。

---

## Phase C: 逐 Skill 构建循环

对每个 skill 重复以下四步：

### Step 1: 创建 `skills/{name}/SKILL.md`

SKILL.md 是 skill 的唯一声明文件，包含 frontmatter 和 replan 指令两部分。

**Frontmatter 字段：**
- `name` — 小写连字符格式（如 `image-editing`）
- `description` — 意图导向描述，>15 字符，说明"用户想做什么"而非"skill 能做什么"
- `type` — 四选一：`generate:image` / `generate:video` / `clarify` / `search`
- `allowed_actions` — 该 skill 可使用的 exec tools 子集

**Body 部分（replan 指令）：**
- 适用场景：skill 处理哪些情况
- 触发条件：何时启动 replan
- 执行规则：replan 中的约束
- 不适用声明：明确边界外的情况

### Step 2: 实现工具函数

- 在 `app/tools/` 添加具体工具实现
- 在 `tool_registry` 注册工具
- 在 `plan_tools.py` 补充 schema 定义

### Step 3: 全链路 curl 测试

验证完整链路：router → plan → load_skill → replan → execute。确保 skill 端到端可运行。

### Step 4: 下一个 Skill

回到 Step 1，重复直到所有 skill 构建完成。

---

## Phase D: 闭合与衔接

### Step 5: 自动生成 `.routing-quality/config.md`

生成 Baseline Intent 表格，兼容路由质量配置模板。每个 skill 一行，包含名称、意图、类型、allowed_actions。其中"必须通过的场景"字段标记为 **"由路由质量调优填充"**，留给后续路由质量调优阶段补全。

### Step 6: 生成路由测试脚手架

生成 `tests/routing_test_{N}skills.py`，每个 skill 包含 3 个初始 golden case（来自 Phase A Baseline Intent）。测试用例使用 4-tuple 格式，资源构造包含完整字段。

### Step 7: 显示衔接状态并提示交接

展示所有产出物状态，提示用户可选安装 **routing-quality-skills** 插件开始 TDD 路由质量调优。

---

## 衔接协议

Phase C/D 的产出与路由质量调优之间通过 7 项衔接协议连接：

| 衔接项 | 前半段产出 | 后半段消费 | 格式要求 |
|--------|-----------|-----------|---------|
| Skill name | frontmatter `name` | `expected_skill` | 小写+连字符 |
| Description | frontmatter `description` | Baseline Intent "一句话意图" | 意图导向，>15 字符 |
| Type | frontmatter `type` | generative 判断 | `generate:image` / `generate:video` / `clarify` / `search` |
| allowed_actions | frontmatter `allowed_actions` | `get_replan_tools()` 过滤 | exec tools 子集 |
| Resource | `resources.py` TypedDict | routing_test 资源构造 | semantic_label / mode / is_latest_result，type 用中文 |
| Config | `.routing-quality/config.md` | 路由质量配置读取 | Baseline Intent 表格 |
| Test cases | `routing_test_{N}skills.py` | 路由质量测试 | 4-tuple |

**衔接要点：**

1. **Skill name** — 贯穿全链路的唯一标识。SKILL.md 中的 `name` 必须与测试中的 `expected_skill` 完全一致。
2. **Description** — 直接影响路由准确率。description 决定 planner 如何理解 skill 意图，同时作为 config.md 中"一句话意图"的来源。
3. **Type** — 决定 skill 是否为 generative 类型，影响路由层对 skill 的分类和竞争判断。
4. **allowed_actions** — 约束 skill 在 replan 阶段可调用的工具集，必须与 `get_replan_tools()` 过滤逻辑匹配。
5. **Resource** — 测试中构造的资源必须与 `resources.py` 的 TypedDict 结构一致，`type` 字段使用中文（"图片"/"视频"）。
6. **Config** — config.md 是路由质量配置的入口，表格格式必须兼容后续自动解析。
7. **Test cases** — 初始 golden case 随 skill 构建同步产出，后续由路由质量调优持续扩充。

---

## 格式规范

### SKILL.md frontmatter

```yaml
---
name: skill-name          # 小写连字符
description: ...          # 意图导向, >15 chars
type: generate:image      # generate:image | generate:video | clarify | search
allowed_actions:
  - respond
  - generate_image
  - load_skill
---
```

### config.md Baseline Intent 表格

```markdown
## Baseline Intent

| Skill名称 | 一句话意图 | 类型 | 必须通过场景(3-5) | 不该走这里(2-3) | 模糊区域 | 竞争skill |
|-----------|-----------|------|------------------|----------------|---------|----------|
| image-editing | 用户想对已有图片进行编辑修改 | generate:image | 由路由质量调优填充 | — | — | — |
```

### 测试 4-tuple

```python
(case_id, asset_desc, user_text, expected_skill)
# 示例：
("G-IMG-01", "用户上传了一张人物照片", "把背景换成海滩", "image-editing")
```

### Resource 构造

```python
{
    "id": "R1-U1",
    "type": "图片",              # 中文: "图片" / "视频"
    "url": "/uploads/xxx.jpg",
    "prompt": "",
    "style": "",
    "mode": "upload",            # upload / generated / ...
    "semantic_label": "用户上传的人物照片",
    "is_latest_result": False
}
```

---

## 产出物清单

| 产出物 | 路径 | 产出阶段 |
|--------|------|---------|
| Skill 定义 | `skills/{name}/SKILL.md` | Step 1 |
| 工具实现 | `app/tools/{name}.py` | Step 2 |
| 工具注册 | `tool_registry` + `plan_tools.py` | Step 2 |
| 路由质量配置 | `.routing-quality/config.md` | Step 5 |
| 路由测试脚手架 | `tests/routing_test_{N}skills.py` | Step 6 |
