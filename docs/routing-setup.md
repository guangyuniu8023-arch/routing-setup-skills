# /routing-setup: 路由 Agent 初始搭建

从零搭建一个具备 skill 路由能力的 Agent 骨架。分两阶段执行：Phase A 收集需求并定义 skill 清单，Phase B 按 9 项架构约束生成目录和代码文件。产出物可直接进入路由质量调优流程。

---

## Phase A: 需求收集

### Step 1: 项目基础信息

向用户收集以下信息：

| 收集项 | 说明 |
|--------|------|
| 项目目标 | 一句话描述 Agent 要解决什么问题 |
| 技术栈 | 语言、框架（FastAPI / Express / ...）、依赖管理 |
| LLM 选择 | 模型供应商 + 模型名称 + 调用方式（SDK / HTTP） |
| 输入类型 | 纯文本 / 文本+图片 / 文本+视频 / 混合 |
| 输出类型 | 文本 / 图片 / 视频 / 混合 |
| Skill 清单 | 让用户列出这个 Agent 需要的所有 skill（可以是初步想法，后续可调整） |

### Step 2: 定义 Skill 清单

逐个 skill 引导用户定义意图，AI 辅助生成草稿：

1. **AI 提问**：对每个 skill 问 —— "用户什么时候应该走 {skill-name}？用一句话描述"
2. **AI 生成草稿**：根据回答推断 3-5 个必须通过场景、2-3 个不该走这里场景、模糊区域和竞争 skill
3. **呈现表格**：

| Skill名称 | 一句话意图 | 类型 | 必须通过场景(3-5) | 不该走这里(2-3) | 模糊区域 | 竞争skill |
|-----------|-----------|------|-------------------|----------------|---------|-----------|
| 小写连字符命名 | 用户的一句话回答（50-300 字符，含核心意图+边界条件） | skill 输出类型（可自定义） | AI 生成，用户确认 | AI 生成，用户确认 | AI 推断，用户补充 | AI 推断，用户确认 |

**场景格式**：每个场景用一句模拟用户输入的自然语言表示，例如：
- 必须通过：`"把这张照片的背景换成海滩"`
- 不该走这里：`"帮我画一只猫"`

这些场景将在路由质量调优中转化为 4-tuple 测试用例 `(case_id, asset_desc, user_text, expected_skill)`。

4. **用户审阅修正**：用户对 AI 草稿逐项确认或修改，确认后锁定

> **TDD 原则**：AI 生成草稿基于用户的口头意图，不读取已有 description。Baseline Intent 是测试种子的来源，必须反映真实用户意图。

命名规范：
- Skill 名称使用**小写连字符**（如 `image-generation`、`subject-clarification`）
- Resource type 使用**中文**（`"图片"` / `"视频"`）

### Step 3: 用户确认

将 Step 1-2 的结果整理后请用户确认，确认后进入 Phase B。

---

## Phase B: Agent 骨架搭建

### Step 4: 生成目录结构

按以下结构创建目录（不含文件内容）：

```
app/
├── main.py                     # FastAPI + session
├── config.py                   # Settings
├── agent/
│   ├── skill_store.py          # 3-tier SkillStore
│   ├── plan_tools.py           # Pydantic + StructuredTool + get_replan_tools()
│   ├── chat_state.py           # ChatState TypedDict
│   ├── planner_prompt.txt      # Planner system prompt
│   ├── replan_prompt.txt       # Replan prompt
│   └── nodes/
│       ├── router_node.py      # Pre-LLM triage
│       └── plan_node.py        # plan_node() + replan_node()
├── services/
│   ├── execute_service.py      # while True + SSE
│   └── action_executor.py      # tool_registry
├── models/
│   └── resources.py            # Resource TypedDict
├── tools/                      # 空目录，Phase C 填充
└── utils/
    └── sse.py                  # make_sse()
skills/                         # Phase C 创建 SKILL.md
```

### Step 5: 用户确认目录

展示目录结构，等待确认后继续。

### Step 6: 按约束生成文件

逐文件生成代码，每个文件必须满足对应的架构约束（见下节）。

### Step 7: 首个 SKILL.md + 冒烟测试

- 用 Step 2 中第一个 skill 生成 `skills/{skill-name}/SKILL.md`
- 冒烟测试格式为 4-tuple：`(case_id, asset_desc, user_text, expected_skill)`
- 验证 SkillStore 能正确扫描、加载该 skill

---

## 9 项架构约束

每项约束标注来源 fix ID，生成代码时必须遵守。

| # | 约束名称 | Fix ID | 要点 |
|---|---------|--------|------|
| 1 | **SkillStore 三层加载** | A1 | `scan()` 读 frontmatter（~30-50 tokens）→ `load()` 读 body → `read_file()` 读附属文件。`get_catalog_text()` 生成 XML。YAML parser 使用正则 `^([a-zA-Z_][a-zA-Z0-9_-]*)\s*:\s*(.*)` 避免冒号 bug |
| 2 | **Schema-only Noop Tools** | A5 | Pydantic schema + `StructuredTool` + `bind_tools()`。区分 `_PLANNER_ONLY_TOOLS`（规划用）与 `_EXEC_TOOLS`（执行用）。`get_replan_tools(allowed_actions)` 按 allowed_actions 过滤 |
| 3 | **Router Node 三路分派** | A2 | no-media-no-text → `require_upload`；has-media-no-text → `clarify`；has-text → `plan`。IMAGE_PATTERN 检测 `/uploads/` 路径 |
| 4 | **Plan Node 结构** | C2, H1 | `create_plan(msg, skill_name, ctx)` → `_tool_calls_to_plan()` → `[respond(msg), load_skill(skill, ctx)]`。每个 step 含 `step_id` + `status:"pending"` |
| 5 | **Replan Node 独立函数** | H3, A3, A4, I1 | 独立于 plan_node；`replan_prompt.txt` 注入 `{skill_content}` + `{allowed_actions_table}`；allowed_actions 过滤；respond-before-generate 过滤；`has_executable` 检查；no-resource → `request_user_upload` |
| 6 | **Execute Service 命令式循环** | -- | `while True` 命令式循环（非 LangGraph StateGraph）；respond hoisting；`load_skill` → break → replan；`last_generate_ok` 标志位；SSE streaming |
| 7 | **Resource 数据结构** | C1, H2 | `{id:"R1-U1", type:"图片"|"视频", url, prompt, style, mode:"upload"|"generate", semantic_label, is_latest_result}` — type 字段使用中文 |
| 8 | **SSE 事件类型** | I3 | `message_delta`/`done`、`stage_started`、`skill_loaded`、`generation_started`/`progress`、`loading_pulse`、`image_generated`/`video_generated`、`user_selection`/`upload_required`、`error`、`done`、`skill_loading_trace` |
| 9 | **Prompt 模板变量** | C3, H4, H5 | planner: `{skill_catalog}` `{resource_summary}` `{memory_summary}`；replan: `{skill_content}` `{allowed_actions_table}` |

**术语说明**（约束中的域特定概念）：
- **respond hoisting**：执行循环中，将"回复用户"动作提前到"生成内容"之前，确保用户先收到文字反馈
- **respond-before-generate 过滤**：replan 产出的动作序列中，如果 respond 排在 generate 后面，则交换顺序
- **last_generate_ok 标志位**：记录上一次生成是否成功，用于决定是否需要 replan
- **has_executable 检查**：验证 replan 产出的 plan 中至少有一个可执行动作，否则降级处理

> **生成指导**：约束描述的是架构方向，不是完整规格。AI 生成代码时应：
> 1. 每个文件先写类/函数签名和核心逻辑注释，展示给用户确认结构
> 2. 用户确认后再填充实现细节
> 3. 域特定概念（如 respond hoisting = 将 respond 动作提到 generate 之前执行；respond-before-generate 过滤 = replan 产出中如果 respond 在 generate 后面则交换顺序）由 AI 根据项目需求适配
> 4. 不强求 1:1 复刻约束描述，允许按项目实际情况调整实现方式

### Step 7.5: 建立测试基础设施

为项目创建路由测试 runner，使后续 TDD 流程可执行：

1. **测试文件**：`tests/routing_test.py`，包含：
   - 测试函数接收 4-tuple `(case_id, asset_desc, user_text, expected_skill)`
   - 调用 planner 获取路由结果
   - 比较 `actual_skill == expected_skill`
   - 输出格式：`PASS/FAIL case_id expected→actual`

2. **测试命令**：记录到 `.routing-quality/config.md` 的 `test_command` 字段，例如：
   ```bash
   python -m pytest tests/routing_test.py -v
   ```

3. **成功标准**：
   - 单个 case：`actual_skill == expected_skill`
   - 整体：通过率 >= 目标阈值（95%+ golden cases）

4. **3-run 多数决**：对每个 case 运行 3 次取多数结果，消除 LLM 随机性

AI 生成测试 runner 代码骨架，用户确认后写入项目。

---

## 产出物

Phase A + B 完成后，交付以下内容：

| 产出物 | 说明 |
|--------|------|
| `app/` 目录 | 所有骨架代码文件，满足 9 项约束 |
| `skills/{first-skill}/SKILL.md` | 首个 skill 描述文件 |
| Baseline Intent 表 | 所有 skill 的意图、正例、反例、竞争关系 |
| 冒烟测试 4-tuple | `(case_id, asset_desc, user_text, expected_skill)` 格式 |
| `.routing-quality/config.md` | 供路由质量调优使用的项目配置 |

完成后可进入 `/routing-skill-build` 构建剩余 skill，或可选安装 routing-quality-skills 插件开始 TDD 路由质量调优。
