# Routing Setup Skills — Flow Guide

本指南覆盖 Agent 搭建的前半段流程：从零开始搭建骨架，逐个构建 skill，直到所有 skill 可运行并产出路由质量配置。

---

## 流程概览

```
/routing-setup → /routing-skill-build → (可选) routing-quality-skills
  Phase A+B       Phase C+D              后半段：TDD 路由质量调优
```

---

## Phase A: 需求收集（/routing-setup）

与用户交互收集以下信息：

1. **项目基本信息** — 目标、技术栈、LLM 选择、输入/输出类型
2. **Skill 清单** — 使用 Baseline Intent 表格定义每个 skill 的名称、意图、类型、正例、反例
3. **用户确认** — 整理摘要后请用户确认

详细规范见 [routing-setup.md](routing-setup.md)。

---

## Phase B: Agent 骨架搭建（/routing-setup）

根据需求收集结果，按 9 项架构约束生成代码：

1. **目录结构** — `app/`、`skills/`、`tests/` 等
2. **逐文件生成** — 每个文件满足对应约束，逐个确认
3. **首个 SKILL.md + 冒烟测试** — 验证 SkillStore scan -> load 链路

9 项架构约束涵盖：SkillStore 三层加载、Schema-only Noop Tools、Router Node 三路分派、Plan Node 结构、Replan Node 独立函数、Execute Service 命令式循环、Resource 数据结构、SSE 事件类型、Prompt 模板变量。

详细规范见 [routing-setup.md](routing-setup.md)。

---

## Phase C: 逐 Skill 构建循环（/routing-skill-build）

对每个 skill 重复以下步骤：

1. **创建 SKILL.md** — frontmatter（name、description、type、allowed_actions）+ replan 指令
2. **实现工具函数** — `app/tools/` 实现 + `tool_registry` 注册 + `plan_tools.py` schema
3. **curl 全链路测试** — router -> plan -> load_skill -> replan -> execute
4. **下一个 skill** — 重复直到所有 skill 完成

详细规范见 [routing-skill-build.md](routing-skill-build.md)。

---

## Phase D: 闭合与衔接（/routing-skill-build）

所有 skill 完成后：

1. **生成 `.routing-quality/config.md`** — Baseline Intent 表格，"必须通过场景"标注为待填充
2. **生成路由测试脚手架** — `tests/routing_test_{N}skills.py`，每 skill 3 个 golden case
3. **展示衔接状态** — 汇总所有产出物，提示可进入路由质量调优

详细规范见 [routing-skill-build.md](routing-skill-build.md)。

---

## 完成后

前半段（Phase A-D）完成后，你已拥有：

- 完整的 Agent 骨架代码（满足 9 项架构约束）
- 所有 skill 的 SKILL.md + 工具实现
- Baseline Intent 配置表
- 路由测试脚手架

**下一步（可选）：** 安装 [routing-quality-skills](https://github.com/guangyuniu8023-arch/routing-quality-skills) 插件，运行 `/routing-layer1` 开始 TDD 路由质量调优。该插件提供 Layer 1-4 四层递进式路由质量工程，从 golden cases 到 integration 全覆盖。
