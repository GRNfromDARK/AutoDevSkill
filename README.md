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
auto-requirement → auto-todo → auto-dev → [Bug Hunt]
(产品决策层)         (工程任务层)   (代码实现层)  (质量保障层)
```

This skill handles **code implementation** — generating automated TDD pipelines from task lists. Product decisions and task decomposition are handled by upstream skills.

## Key Features

- **Spec-anchored TDD** — 5-step closure per card: RED → GREEN → SPEC → LINT → RUN
- **Automated gate checks** — Unit tests, regression, decision audit, cross-file coverage at phase boundaries
- **Auto-repair** — Up to 10 retries per card on test failure
- **Independent acceptance verification** — Separate AI verifies each acceptance criterion
- **3-level decision protocol** — SPEC-DECISION (self), AI-REVIEW (peer), AI-GATE (phase boundary)
- **Decision audit trail** — All decisions logged to `decisions.jsonl` for cross-card traceability
- **Resumable execution** — State file tracks progress; resume from any card with `--from`
- **Bug Hunt phase** — multi-round P0/P1/P2 bug scanning after development, with auto-fix + regression tests

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
├── bug_reports/        # Bug Hunt reports per round
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

### v1.4 (2026-03-11)

- **Feature**: Bug Hunt phase — after all cards complete and summary.md is generated, a multi-round bug scanning + fixing loop runs automatically:
  1. **Bug Scan** (VERIFY_MODEL, read-only auditor) — scans all changed code against original requirements, finds P0/P1/P2 bugs
  2. **Bug Fix** (MODEL, developer) — fixes all reported bugs + writes regression tests for each
  3. **Test Verify** — runs full test suite with auto-repair (reuses `TEST_MAX_RETRIES`)
  4. **Fix Verify** (VERIFY_MODEL, auditor) — independently verifies each fix + regression test quality
  5. Loops until scanner reports `NO_BUGS_FOUND` or max rounds reached
  - Inner fix-verify retries reuse `AC_MAX_RETRIES` (default 10)
  - Outer scan rounds: `{ENV_PREFIX}_BUG_HUNT_ROUNDS` (default 15)
  - Bug reports saved to `bug_reports/bug_report_round_N.md`
  - Results appended to `summary.md` at completion
- **Prompt quality**: 5 prompt optimizations for Bug Hunt agents:
  - Summary Update uses append mode (prevents data loss from AI hallucination)
  - Bug Scan injects previous rounds' reports (prevents re-reporting fixed bugs)
  - Bug Fix includes verify feedback on retries + regression test placement guidance
  - Fix Verify checks test quality (not just existence) + supports PARTIAL status
  - Auditor prompts enforce read-only constraint; Test Fix includes bug context
- **Robustness**: stderr separated from AI response in Bug Scan and Fix Verify (prevents verbose noise in reports); `--reset` clears `bug_reports/`; `phase_gate.md` excluded from requirements context; Bug Hunt state tracked in state file (`BUG_HUNT_ROUND:N` + `BUG_HUNT_DONE`) for crash recovery

### v1.3 (2026-03-09)

- **Fix**: Acceptance verification (AC) retry limit increased from 3 to 10 — independent acceptance verifier now has more attempts to self-heal before failing. Configurable via `{ENV_PREFIX}_AC_RETRIES` env var (default: 10).
- **Fix**: Gate check retry limit increased from 3 to 10 — gate script (lint/format/type-check) now has more attempts to auto-fix before failing. Configurable via `{ENV_PREFIX}_GATE_RETRIES` env var (default: 10).

### v1.2 (2026-03-06)

- **Fix**: Test retry limit increased from 3 to 10 — template was incorrectly reusing `GATE_MAX_RETRIES` (default 3) for the test loop instead of a dedicated `TEST_MAX_RETRIES` variable. Now configurable via `{ENV_PREFIX}_TEST_RETRIES` env var (default: 10).
- **Fix**: Interactive test blocking — pipelines could hang indefinitely when the test command included interactive tests (e.g., wallet tests requiring private key input via `input()`). Added three layers of protection:
  1. **Generation-time**: Step 2 now requires scanning test directories for interactive tests (`input()`, `getpass()`, wallet prompts, manual e2e) and adding `--ignore` flags to exclude them.
  2. **Runtime**: Test execution wrapped with 300s timeout via `$TIMEOUT_CMD`; exit code 124 triggers immediate failure with descriptive error message instead of hanging forever.
  3. **Principles**: Added "Test command must be non-interactive" as Key Principle #4.

### v1.1 (2026-03-05)

- **BREAKING**: Default model for all AI calls changed to `opus` (including verification). Override with `{ENV_PREFIX}_VERIFY_MODEL` env var if needed.
- **Fix**: Heredoc variable expansion — dynamic content (`$test_output`, `$verify_output`, `$card_content`, etc.) containing shell metacharacters (`$`, `` ` ``, `\`, ANSI escapes) was silently destroyed by unquoted heredoc expansion. Now uses temp files with `printf '%s\n'` for safe injection.
- **Fix**: `echo "$prompt" | claude` replaced with `claude < tmpfile` to avoid `ARG_MAX` failures on large prompts (test output, card content, AI verbose output).
- **Fix**: Gate fix, AC fix, and summary prompts all migrated to the same safe temp-file pattern.

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

## Bug Hunt 阶段

开发完成后自动进入 Bug Hunt，确保交付可运行的代码：

```
所有 Card 完成 → summary.md → Bug Hunt 多轮循环 → 最终交付
```

| 步骤 | 角色 | 说明 |
|------|------|------|
| Bug Scan | 独立审计员（只读） | 扫描代码 + 对照需求，找出 P0/P1/P2 bug |
| Bug Fix | 开发者 AI | 修复 bug + 编写回归测试 |
| 测试验证 | 自动 | 运行全量测试，失败自动修复 |
| Fix Verify | 独立审计员（只读） | 验证修复 + 检查回归测试质量 |

- 无新 bug → 退出循环
- 默认最多 15 轮（`{ENV_PREFIX}_BUG_HUNT_ROUNDS`）
- 支持断点续跑（crash 后从上次完成的轮次继续）
