# auto-dev

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill plugin that generates complete **Autodev automated development pipelines** from a structured `todolist.md` file. Batteries included — all dependency skills bundled, install once, everything works.

## What is Autodev?

Autodev is a **spec-anchored, gate-guarded, TDD-enforced automated development methodology**:

1. Splits a task list into sequential **Task Cards** (one AI session per card)
2. Enforces **TDD 5-step closure** per card (RED → GREEN → SPEC → LINT → RUN)
3. Runs **automated gate checks** between phases
4. **Auto-repairs** test failures (up to 3 retries per card)
5. Tracks progress in a **state file** for resumable execution
6. **Independent acceptance verification** per card (separate AI verifies ACs)
7. **AI mutual review** for high-risk decisions (3-level decision protocol)
8. **Decision audit trail** (`decisions.jsonl`) for cross-card traceability
9. **Pipeline completion summary** — auto-generated report on finish

## How It Works

Give it a `todolist.md` → the skill reads it and generates a self-contained pipeline:

```
todolist.md  →  Autodev/{project}/
                 ├── autodev.sh          # Main pipeline script
                 ├── system_prompt.md    # AI session prompt
                 ├── gate_check.sh       # Automated gate checks
                 ├── state               # Progress tracker
                 ├── decisions.jsonl     # Decision audit trail
                 ├── logs/               # Runtime logs
                 └── cards/
                     ├── A.1.md          # Task cards
                     ├── A.2.md
                     ├── B.1.md
                     └── phase_gate.md   # Phase gate audit template
```

Then run:

```bash
./Autodev/{project}/autodev.sh
```

The pipeline drives Claude Code through card-by-card development — fully automated, no human in the loop.

## Generator Process

The skill follows 5 steps to generate the pipeline:

1. **Read the todolist** — extract groups, tasks, dependencies, test commands, constraints
2. **Gather project context** — identify spec doc, source files, test paths, key constraints
3. **Design the pipeline** — map groups → phases → cards, define gate checks
4. **Generate all files** — autodev.sh, system_prompt.md, gate_check.sh, cards/*.md, etc.
5. **Verify** — dry-run the script, confirm all card files exist, no unresolved placeholders

## Pipeline Workflow

```
┌─────────────────────────────────────────────┐
│  Phase A                                    │
│  Card A.1 → Card A.2 → ... → GATE:A        │
├─────────────────────────────────────────────┤
│  Phase B                                    │
│  Card B.1 → Card B.2 → ... → GATE:B        │
├─────────────────────────────────────────────┤
│  ...                                        │
│  Card N.1 → ... → GATE:FINAL               │
└─────────────────────────────────────────────┘
                    ↓
           summary.md (auto-generated)
```

**Each Card** runs one Claude Code session:
1. Reads spec + todolist + existing code
2. Implements via TDD (test first → implement → verify)
3. Auto-repairs if tests fail (up to 3 retries)
4. Independent AI verifies acceptance criteria (separate model)
5. Records decisions to `decisions.jsonl`

**Each Gate** runs automated checks:
1. Unit tests + full regression
2. Decision audit (SPEC-DECISION / AI-REVIEW annotations)
3. Cross-file change coverage
4. AI compliance audit

## 3-Level Decision Protocol

The pipeline runs fully unattended. All decisions that would normally require human confirmation are handled by AI mutual review:

| Level | Name | When | How |
|-------|------|------|-----|
| L1 | **SPEC-DECISION** | Low-risk: naming, style, small impl choices | AI self-decides with 3-round analysis |
| L2 | **AI-REVIEW** | High-risk: architecture, backward compat, cross-file changes | Spawn independent review agent |
| L3 | **AI-GATE** | Phase boundaries | gate_check.sh + AI audit session |

All decisions are recorded in `decisions.jsonl` for cross-card traceability.

## Install

```bash
npx skills add GRNfromDARK/AutoDevSkill
```

This installs **auto-dev** and 7 bundled dependency skills:

| Skill | Role in Pipeline |
|-------|-----------------|
| `auto-dev` | Core pipeline generator |
| `test-driven-development` | TDD workflow: write test first, watch it fail, implement |
| `systematic-debugging` | Root cause investigation when tests fail |
| `verification-before-completion` | Evidence-based completion checks before claiming done |
| `dispatching-parallel-agents` | Parallel execution for cards with independent files |
| `brainstorming` | Design exploration (adapted for unattended AI-REVIEW) |
| `subagent-driven-development` | Subagent task dispatch with spec + quality review |
| `requesting-code-review` | Code review workflow via review agents |

Dependency skills are from [obra/superpowers](https://github.com/obra/superpowers).

## Usage

In Claude Code:

```
帮我生成 probability_engine 的 autodev 文件
```

```
create autodev for my-feature based on todolist.md
```

```
generate autodev pipeline
```

## CLI Options

```
./Autodev/{project}/autodev.sh [OPTIONS]

--from CARD_ID    Resume from a specific card (e.g. --from B.1)
--model MODEL     Claude model (default: opus)
--reset           Clear all progress, start fresh
--dry-run         Show execution plan without running
--status          Show current progress
--help            Show help
```

## Requirements

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed
- A `todolist.md` with grouped tasks
- Optionally, a spec/design document referenced by the todolist

---

## 中文使用说明

### 安装

```bash
npx skills add GRNfromDARK/AutoDevSkill
```

安装后会自动获得 `auto-dev` 和 7 个依赖 skill，无需额外配置。

### 前置准备

1. 确保已安装 [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code)
2. 准备好一份 `todolist.md`，按分组列出任务（Group 1、Group 2...）
3. （可选）准备设计文档 / spec 文档，作为开发的"唯一真相源"

### 生成流水线

在 Claude Code 中说：

```
帮我生成 my_project 的 autodev 文件
```

Skill 会自动：
1. 读取你的 `todolist.md`，提取分组、任务、依赖关系、测试命令
2. 收集项目上下文（spec 文档、源文件、约束条件）
3. 将分组映射为阶段（Phase），将任务映射为卡片（Card）
4. 生成所有文件到 `Autodev/{project}/` 目录
5. 自动验证（dry-run、检查占位符、确认文件完整）

### 运行流水线

```bash
# 启动完整流水线
./Autodev/{project}/autodev.sh

# 查看执行计划（不实际运行）
./Autodev/{project}/autodev.sh --dry-run

# 查看当前进度
./Autodev/{project}/autodev.sh --status

# 从指定 Card 恢复执行
./Autodev/{project}/autodev.sh --from B.1

# 切换模型
./Autodev/{project}/autodev.sh --model sonnet

# 清除进度，从头开始
./Autodev/{project}/autodev.sh --reset
```

### 流水线执行过程

每张 **Card**（任务卡片）的执行流程：

```
1. 读取 spec + todolist + 已有代码
2. 按 TDD 流程实现（RED → GREEN → SPEC → LINT → RUN）
3. 测试失败 → AI 自动修复（最多 3 次重试）
4. 独立 AI 验收审计（用轻量模型逐条检查验收标准）
5. 验收未通过 → AI 自动修复 → 重新验收（最多 3 轮）
6. 记录决策到 decisions.jsonl
```

每个 **Gate**（阶段门禁）的检查项：

```
1. 单元测试 + 全量回归测试
2. 决策审计（SPEC-DECISION / AI-REVIEW 标注统计）
3. 跨文件变更覆盖率检查
4. AI 合规审计
```

### todolist.md 格式建议

```markdown
## Group 1: 基础模块

- [ ] 任务 1.1: 实现 XXX 核心逻辑
- [ ] 任务 1.2: 添加 YYY 数据验证
- [ ] 任务 1.3: 编写单元测试

## Group 2: 集成层

- [ ] 任务 2.1: 对接 API 接口
- [ ] 任务 2.2: 添加错误处理
- [ ] 任务 2.3: 集成测试

## 测试命令
pytest tests/ -q

## 约束
- 向后兼容现有 API
- 不修改数据库 schema
```

### 三级决策协议

流水线全程无人值守，所有需要人类确认的场景由 AI 互审替代：

| 级别 | 名称 | 适用场景 | 处理方式 |
|------|------|----------|----------|
| L1 | **SPEC-DECISION** | 参数命名、代码风格、小范围实现选择 | AI 三轮自决（列举方案 → 推演影响 → 选择最优） |
| L2 | **AI-REVIEW** | 架构变更、向后兼容、跨文件接口修改 | 启动独立 review agent 互审 |
| L3 | **AI-GATE** | 阶段边界（Phase Gate） | gate_check.sh 自动检查 + AI 审计会话 |

所有决策记录在 `decisions.jsonl` 中，后续 Card 可自动获取前序决策上下文。

## Project Structure

```
AutoDevSkill/
├── README.md
└── skills/
    ├── auto-dev/                          # Core skill
    ├── test-driven-development/           # Bundled dependency
    ├── systematic-debugging/              # Bundled dependency
    ├── verification-before-completion/    # Bundled dependency
    ├── dispatching-parallel-agents/       # Bundled dependency
    ├── brainstorming/                     # Bundled dependency
    ├── subagent-driven-development/       # Bundled dependency
    └── requesting-code-review/            # Bundled dependency
```

## Credits

Bundled dependency skills are from [obra/superpowers](https://github.com/obra/superpowers) by Jesse Vincent.

## License

MIT
