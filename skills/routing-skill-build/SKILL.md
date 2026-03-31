---
name: routing-skill-build
description: 逐个构建 Agent 的 skill（SKILL.md + 工具实现 + 测试），完成后自动生成路由质量配置和测试脚手架
type: guide
trigger: Agent 骨架已搭建（/routing-setup 完成），需要逐个填充 skill
---

# Routing Skill Build

本技能覆盖 Phase C（逐 skill 构建循环）和 Phase D（闭合与衔接路由质量层）。

## 前置条件

- `/routing-setup` 已完成（有 `app/` 目录、Baseline Intent 表、首个 SKILL.md）

---

## Phase C: 逐 Skill 构建循环

对每个 skill 重复 Step 1-4，完成一个再进入下一个。

### Step 1: 创建 `skills/{name}/SKILL.md`

frontmatter 必填字段：

| 字段 | 规则 |
|------|------|
| `name` | 小写连字符，如 `image-generation` |
| `description` | 意图导向，50-300 字符。描述用户想做什么而非 skill 能做什么。应包含核心意图 + 关键边界条件（什么情况不该走这里） |
| `type` | skill 的输出类型。常见值：`generate:image` / `generate:video` / `generate:text` / `clarify` / `search` / `analyze`，也可自定义（如 `diagnose`、`recommend`） |
| `allowed_actions` | 该 skill 可调用的 action 子集。从 `app/agent/plan_tools.py` 的 `_EXEC_TOOLS` 列表中选取适用项 |

body 部分的 replan 指令由 AI 生成草稿，用户审阅确认：

1. AI 读取该 skill 的 frontmatter（name、type、allowed_actions）+ Baseline Intent 中的意图和场景
2. AI 生成 replan 指令草稿，至少包含以下内容（可按需增加其他段落如输入格式、输出格式、示例等）：
   - **适用场景** — 什么情况下进入该 skill 的 replan
   - **触发条件** — 具体判断逻辑
   - **执行规则** — replan 时的约束
   - **不适用声明** — 明确不处理哪些情况（防路由越界）
3. 用户审阅并修正草稿，可增加项目特定的段落
4. 确认后写入 SKILL.md body

### Step 2: 实现工具函数

- 在 `app/tools/` 下实现具体工具函数
- 在 `tool_registry` 中注册工具
- 在 `plan_tools.py` 中添加对应 schema

### Step 3: curl 全链路测试

测试完整链路：router -> plan -> load_skill -> replan -> execute

```bash
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "测试用例文本", "session_id": "test-xxx"}'
```

测试失败时必须修复后重新测试，不允许跳过。

### Step 4: 下一个 Skill

回到 Step 1，直到所有 skill 构建完毕。

---

## Phase D: 闭合与衔接

所有 skill 完成后执行以下步骤。

### Step 5: 自动生成 `.routing-quality/config.md`

AI 从所有已构建的 SKILL.md frontmatter 自动提取信息，生成 Baseline Intent 表。用户只需确认表格完整性。格式：

```markdown
| Skill名称 | 一句话意图 | 类型 | 必须通过场景(3-5) | 不该走这里(2-3) | 模糊区域 | 竞争skill |
|-----------|-----------|------|------------------|----------------|---------|----------|
| image-generation | 用户想根据文字描述生成全新图片 | generate:image | 1. "画一只猫" 2. "生成赛博朋克风城市" 3. "帮我画一张海报" | 1. "把这张照片变清晰"(editing) 2. "这张图是什么风格"(search) | 有参考图的生成 vs editing | image-editing |
```

config.md 携带 Phase A 已确认的必须通过场景和不该走这里场景。路由质量调优在此基础上扩展（从 3-5 个扩到更多），而不是从零重建。

### Step 6: 生成 `tests/routing_test_{N}skills.py`

测试用例采用 4-tuple 格式：

```python
(case_id, asset_desc, user_text, expected_skill)
```

每个 skill 从 Baseline Intent 中提取 3 个 golden case。

资源构造必须包含以下字段：
- `semantic_label` - 资源语义标签
- `mode` - 资源模式
- `is_latest_result` - 是否为最新生成结果
- `type` - 资源类型（中文）

### Step 7: 展示衔接状态

输出所有 skill 的构建状态汇总，然后提示：

> 所有 skill 构建完成。可选安装 **routing-quality-skills** 插件并运行 `/routing-layer1` 开始 TDD 路由质量调优。

---

## 执行规则

1. 每个 skill 的 Step 1-3 完成后才进入下一个 skill
2. Step 3 curl 测试失败时，修复后重新测试，不跳过
3. Phase D 在所有 skill 完成后才执行

---

## 衔接协议（7 项）

| # | 从 | 到 | 说明 |
|---|----|----|------|
| 1 | Skill name | expected_skill | 小写连字符，测试用例直接引用 |
| 2 | Description | Baseline Intent "一句话意图" | 意图导向，50-300 字符，含核心意图+边界条件 |
| 3 | Type | generative 判断 | 常见值：`generate:image` / `generate:video` / `generate:text` / `clarify` / `search` / `analyze`，可自定义 |
| 4 | allowed_actions | get_replan_tools() 过滤 | 控制 replan 可用工具集 |
| 5 | Resource | routing_test 资源构造 | type 中文，semantic_label / mode / is_latest_result |
| 6 | Config | 路由质量配置读取 | Baseline Intent 表格格式一致 |
| 7 | Test cases | 路由质量测试 | 4-tuple 格式贯穿全部层级 |

详细规范见 [docs/routing-skill-build.md](../../docs/routing-skill-build.md)。
