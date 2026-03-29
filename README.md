# Routing Setup Skills

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

Front half of routing engineering — scaffold a Plan-Execute routing Agent from scratch and build each skill with SKILL.md, tools, and tests.

Works with **any planner + skill architecture** where an LLM routes user intent to skill descriptions. Platform-agnostic: usable in Claude Code, Cursor, Windsurf, Gemini CLI, Codex CLI, or any tool that supports [SKILL.md](https://agentskills.io).

## Pipeline

```
/routing-setup → /routing-skill-build → (handoff to routing quality)
  scaffold        per-skill build        optional: routing-quality-skills
├────── This plugin ────────┤
```

| Skill | Phase | What happens |
|-------|-------|-------------|
| `/routing-setup` | A + B | Collect requirements, define skill inventory, generate Agent skeleton with 9 architecture constraints, create first SKILL.md + smoke test |
| `/routing-skill-build` | C + D | Build each remaining skill (SKILL.md + tools + curl test), generate `.routing-quality/config.md` and routing test scaffold |

## Quick Start

### 1. Claude Code (Plugin mode)

```bash
git clone https://github.com/guangyuniu8023-arch/routing-setup-skills.git

claude --plugin-dir ./routing-setup-skills
```

Both skills are immediately available as slash commands:

```
/routing-setup          # Start here — scaffold Agent from scratch
/routing-skill-build    # Build each skill one by one
```

### 2. Cursor

```bash
git clone https://github.com/guangyuniu8023-arch/routing-setup-skills.git

# Copy skills into Cursor's skill directory
cp -r routing-setup-skills/skills/* .cursor/skills/

# Copy docs into your project (skills reference these docs)
cp -r routing-setup-skills/docs your-project/docs/routing-setup/
```

Then type `/routing-setup` in Cursor chat.

### 3. Windsurf

```bash
git clone https://github.com/guangyuniu8023-arch/routing-setup-skills.git

# Copy skills and docs into your project
cp -r routing-setup-skills/skills .windsurf/skills/
cp -r routing-setup-skills/docs your-project/docs/routing-setup/
```

Use `/routing-setup` in Windsurf chat.

### 4. Gemini CLI

```bash
git clone https://github.com/guangyuniu8023-arch/routing-setup-skills.git

# Copy into your Gemini skills directory
cp -r routing-setup-skills/skills ~/.gemini/skills/
cp -r routing-setup-skills/docs your-project/docs/routing-setup/
```

### 5. Codex CLI / Other AI tools

SKILL.md is an [open standard](https://agentskills.io). Copy `skills/` to wherever your tool loads skills from, and place `docs/` in your project:

```bash
git clone https://github.com/guangyuniu8023-arch/routing-setup-skills.git

# Adapt paths to your tool
cp -r routing-setup-skills/skills /path/to/your/tool/skills/
cp -r routing-setup-skills/docs your-project/docs/routing-setup/
```

### 6. Manual (no tool integration)

You can use the methodology manually by reading the docs directly:

1. Read `docs/README.md` for the flow guide
2. Follow the relevant doc (e.g., `docs/routing-setup.md`) step by step
3. Apply the instructions to your project using any LLM assistant

## Usage

### Path A: Scaffold the Agent

If you don't have an Agent yet, start with `/routing-setup`:

```
/routing-setup
```

What happens:
- **Phase A**: Collects your project goals, tech stack, LLM choice, and skill inventory via Baseline Intent table
- **Phase B**: Generates a complete project structure following 9 architecture constraints (SkillStore 3-tier loading, Router Node, Plan/Replan nodes, Execute Service, Resource model, SSE events, Prompt templates)
- Creates first SKILL.md and a smoke test to verify the skeleton works

### Path B: Build each skill

After the skeleton is ready, run `/routing-skill-build`:

```
/routing-skill-build
```

For each skill in your inventory:
- Creates `skills/{name}/SKILL.md` with frontmatter (name, description, type, allowed_actions) and replan instructions
- Implements tool functions and registers them
- Runs a full-chain curl test (router -> plan -> load_skill -> replan -> execute)

After all skills are built:
- Auto-generates `.routing-quality/config.md` (Baseline Intent table)
- Generates `tests/routing_test_{N}skills.py` (3 golden cases per skill)

### After completing the front half

Once all skills are built and tested, you can optionally install [routing-quality-skills](https://github.com/guangyuniu8023-arch/routing-quality-skills) (the back half) for TDD-based routing accuracy improvement. That plugin provides `/routing-layer1` through `/routing-layer4` and `/fixing-routing` for systematic routing quality engineering.

## Project Structure

```
routing-setup-skills/
├── .claude-plugin/
│   └── plugin.json              # Claude Code plugin metadata
├── skills/
│   ├── routing-setup/           # Phase A+B: Agent skeleton scaffold
│   │   └── SKILL.md
│   └── routing-skill-build/     # Phase C+D: Per-skill build + handoff
│       └── SKILL.md
├── docs/
│   ├── README.md                # Flow guide for the front half
│   ├── routing-setup.md         # 9 architecture constraints detail
│   └── routing-skill-build.md   # Handoff protocol (7 items)
├── .gitignore
├── LICENSE
└── README.md                    # This file
```

## How It Works

Each SKILL.md file uses a standard format with YAML frontmatter + markdown body:

```yaml
---
name: routing-setup
description: "Guides the user to scaffold..."
type: guide
---

# Instructions for the AI assistant
...
```

When you type `/routing-setup` in your AI tool:
1. The tool finds the matching SKILL.md by name
2. Loads the full content (frontmatter + body) into the AI's context
3. The AI follows the instructions in the body, referencing docs as needed

This is why the plugin works across different tools — it's just structured markdown that any LLM can follow.

## Requirements

No prerequisites. This plugin starts from zero and guides you through building an Agent from scratch.

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/my-improvement`)
3. Make your changes
4. Submit a pull request

When modifying skills:
- Keep SKILL.md body < 800 words
- Keep docs < 1000 words
- Follow the [SKILL.md standard](https://agentskills.io)
- Ensure cross-file terminology consistency (skill names use lowercase-hyphen, Resource type uses Chinese)

## License

[MIT](LICENSE)
