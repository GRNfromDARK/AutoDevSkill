# Auto-Dev Skill

Automated TDD pipeline generator for [Claude Code](https://claude.com/claude-code). Generates complete spec-anchored, gate-guarded development pipelines from a `todolist.md` file. Batteries included — all 7 dependency skills bundled.

## Install

```bash
npx skills add GRNfromDARK/AutoDevSkill --all
```

Or install globally:

```bash
npx skills add GRNfromDARK/AutoDevSkill --all -g
```

This installs **auto-dev** and 7 bundled dependency skills:

| Skill | Role |
|-------|------|
| `test-driven-development` | TDD: write test first, watch it fail, implement |
| `systematic-debugging` | Root cause investigation when tests fail |
| `verification-before-completion` | Evidence-based completion checks |
| `dispatching-parallel-agents` | Parallel execution for independent cards |
| `brainstorming` | Design exploration (adapted for AI-REVIEW) |
| `subagent-driven-development` | Subagent dispatch with spec + quality review |
| `requesting-code-review` | Code review via independent review agents |

Dependency skills are from [obra/superpowers](https://github.com/obra/superpowers) by Jesse Vincent.

## Usage

Trigger in Claude Code with any of these keywords:

```
帮我生成 my-project 的 autodev 文件
create autodev for my-feature based on todolist.md
generate autodev pipeline
```

## Pipeline Positioning

```
auto-requirement → auto-todo → auto-dev
(产品决策层)         (工程任务层)   (代码实现层)
```

This skill handles **code implementation** — generating automated TDD pipelines from task lists. Product decisions and task decomposition are handled by upstream skills.

## Key Features

- **Spec-anchored TDD** — 5-step closure per card: RED → GREEN → SPEC → LINT → RUN
- **Automated gate checks** — Unit tests, regression, decision audit, cross-file coverage at phase boundaries
- **Auto-repair** — Up to 3 retries per card on test failure
- **Independent acceptance verification** — Separate AI verifies each acceptance criterion
- **3-level decision protocol** — SPEC-DECISION (self), AI-REVIEW (peer), AI-GATE (phase boundary)
- **Decision audit trail** — All decisions logged to `decisions.jsonl` for cross-card traceability
- **Resumable execution** — State file tracks progress; resume from any card with `--from`

## How It Works

```
Read todolist → Gather project context → Design pipeline → Generate files → Verify
```

## Output

Self-contained pipeline directory:

```
Autodev/{project}/
├── autodev.sh          # Main pipeline script
├── system_prompt.md    # AI session prompt
├── gate_check.sh       # Automated gate checks
├── state               # Progress tracker
├── decisions.jsonl     # Decision audit trail
├── logs/               # Runtime logs
└── cards/
    ├── A.1.md          # Task cards
    └── phase_gate.md   # Phase gate audit template
```

Run with: `./Autodev/{project}/autodev.sh`

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

## 3-Level Decision Protocol

The pipeline runs fully unattended. All decisions handled by AI mutual review:

| Level | Name | When | How |
|-------|------|------|-----|
| L1 | **SPEC-DECISION** | Low-risk: naming, style, small impl choices | AI self-decides with 3-round analysis |
| L2 | **AI-REVIEW** | High-risk: architecture, backward compat, cross-file changes | Spawn independent review agent |
| L3 | **AI-GATE** | Phase boundaries | gate_check.sh + AI audit session |

All decisions recorded in `decisions.jsonl` for cross-card traceability.

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

## Changelog

### v1.01 (2026-02-27)

- Minor improvements and fixes

### v1.0 (2026-02-27)

- Initial release with spec-anchored TDD pipeline, 3-level decision protocol, automated gate checks, independent acceptance verification, and decision audit trail

## License

MIT

---

# 中文使用说明

## 安装

```bash
npx skills add GRNfromDARK/AutoDevSkill --all -g
```

安装后自动获得 auto-dev 和 7 个依赖 skill，无需额外配置。

## 使用

在 Claude Code 中说：

```
帮我生成 my_project 的 autodev 文件
```

## 前置准备

1. 确保已安装 [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code)
2. 准备好一份 `todolist.md`，按分组列出任务
3. （可选）准备设计文档作为开发的"唯一真相源"

## todolist.md 格式

```markdown
## Group 1: 基础模块

- [ ] 任务 1.1: 实现 XXX 核心逻辑
- [ ] 任务 1.2: 添加 YYY 数据验证

## Group 2: 集成层

- [ ] 任务 2.1: 对接 API 接口
- [ ] 任务 2.2: 添加错误处理

## 测试命令
pytest tests/ -q

## 约束
- 向后兼容现有 API
- 不修改数据库 schema
```

## 运行流水线

```bash
./Autodev/{project}/autodev.sh           # 启动
./Autodev/{project}/autodev.sh --dry-run # 预览执行计划
./Autodev/{project}/autodev.sh --status  # 查看当前进度
./Autodev/{project}/autodev.sh --from B.1 # 从指定 Card 恢复
./Autodev/{project}/autodev.sh --reset   # 清除进度重新开始
```

## 三级决策协议

| 级别 | 名称 | 适用场景 | 处理方式 |
|------|------|----------|----------|
| L1 | SPEC-DECISION | 命名、风格、小范围实现 | AI 三轮自决 |
| L2 | AI-REVIEW | 架构、兼容性、跨文件修改 | 独立 review agent |
| L3 | AI-GATE | 阶段边界 | gate_check.sh + AI 审计 |

所有决策记录在 `decisions.jsonl` 中，后续 Card 可自动获取前序决策上下文。
