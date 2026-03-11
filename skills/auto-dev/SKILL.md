---
name: auto-dev
description: Use when user says "帮我生成 autodev", "create autodev for", "generate autodev pipeline", or needs to turn a todolist.md into an automated gated TDD development pipeline.
---

# Autodev Pipeline Generator

## Overview

This skill generates a complete **Autodev automated development pipeline** from a structured todolist.md file. The output is a self-contained directory that runs `autodev.sh` to drive Claude Code through a gated, TDD-enforced, card-by-card development workflow — fully automated, no human in the loop.

**Trigger phrases**: "帮我生成 xxx 的 autodev 文件", "create autodev for", "generate autodev pipeline"

## What is Autodev?

Autodev is a **spec-anchored, gate-guarded, TDD-enforced automated development methodology**. It:

1. Splits a task list into sequential **Task Cards** (one AI session per card)
2. Enforces **TDD 5-step closure** per card (RED → GREEN → SPEC → LINT → GATE)
3. Runs **automated gate checks** between phases
4. **Auto-repairs** test failures (up to 10 retries per card)
5. Tracks progress in a **state file** for resumable execution
6. **Independent acceptance verification** per card (separate AI verifies ACs)
7. **AI mutual review** for high-risk decisions (SPEC-DECISION / AI-REVIEW / AI-GATE 三级)
8. **Decision audit trail** (`decisions.jsonl`) for cross-card traceability
9. **Pipeline completion summary** — auto-generates structured report: what was implemented, files changed, decisions made, test results
10. **Bug Hunt phase** — multi-round bug scanning + fixing after development: independent auditor finds P0/P1/P2 bugs, developer AI fixes with regression tests, loops until clean (up to 15 rounds)

## Checklist

You MUST complete these steps in order:

1. **Read the todolist** — understand groups, tasks, phases, dependencies, test commands
2. **Gather project context** — identify spec doc, source files, test paths, key constraints
3. **Design the pipeline** — map todolist groups → phases → cards, define gate checks
4. **Generate all files** — autodev.sh, system_prompt.md, gate_check.sh, cards/*.md, phase_gate.md, state, decisions.jsonl
5. **Verify** — dry-run the script, confirm all card files exist, paths resolve correctly

## Step 1: Read the Todolist

Read the todolist.md file completely. Extract:

| Item | What to look for |
|------|-----------------|
| **Groups** | Top-level sections (Group 1, Group 2...) — these become Phases |
| **Tasks** | Individual numbered items within groups — these become Cards (or get merged into Cards) |
| **Dependencies** | Which tasks depend on others — determines Card ordering |
| **Execution phases** | If the todolist defines phases (Phase A, B, C...) — use them directly |
| **Test commands** | How to run tests — becomes the test verification step |
| **Key constraints** | Backward compatibility, safety rules, P1 fixes — become system_prompt constraints |
| **Source files** | Which files are being modified — becomes Card "read existing code" sections |
| **Spec/design docs** | Referenced design documents — becomes the "single source of truth" |

## Step 2: Gather Project Context

Resolve context autonomously (no human confirmation in runtime):

```
1. What is the spec/design document path?
   → This is the "single source of truth" referenced by all cards

2. What is the test command?
   → e.g., "python3 -m pytest tests/test_foo.py -q" or "npm test"

3. Are there any interactive/manual tests that must be excluded?
   → CRITICAL: Scan the test directories for tests that require interactive input
     (stdin prompts, private keys, manual confirmation, GUI interaction, network
     services that won't be available in CI). These MUST be excluded from the test
     command via --ignore or path filtering.
   → Common patterns to look for:
     - input(), getpass(), prompt(), readline() calls in test files
     - Tests under directories named: live/, manual/, interactive/, e2e/, integration/
     - Tests that connect to external wallets, require API keys at runtime, or need
       running services
   → Add --ignore flags for each such directory/file. The generated test command must
     be fully non-interactive and able to run unattended.

4. What are the core source files being modified?
   → Listed in system_prompt as "project files"

5. What are the project-specific constraints?
   → e.g., "backward compatible", "no breaking changes", "量纲一致性"

6. Where should the Autodev directory live?
   → Convention: Autodev/{project_name}/

7. If anything is unclear, how should ambiguity be resolved?
   → Infer from repo evidence (todolist, spec, tests, git history, existing patterns),
     choose the most conservative backward-compatible option, and record assumptions in
     system_prompt.md ("Assumptions & Defaults"). Do NOT block for user confirmation.
```

## Step 3: Design the Pipeline

### Mapping Groups to Phases

Each **Group** in the todolist becomes a **Phase**. Each Phase ends with a **GATE**.

```
Group 1 → Phase A → GATE:A
Group 2 → Phase B → GATE:B
...
Last Group → Phase N → GATE:FINAL
```

### Mapping Tasks to Cards

**Merge small related tasks** into single Cards. **Keep complex tasks** as their own Card. Target: **2-5 tasks per Card**, **3-7 Cards per Phase**.

Heuristics for merging:
- Same source file → merge
- Sequential dependency with no external dependency → merge
- Total estimated complexity < 1 Claude session → merge

Heuristics for splitting:
- Different source files → separate
- Independent testable unit → separate
- High complexity (e.g., "MC simulation core") → own Card

### Card ID scheme

Use `{Phase}.{Sequence}` format:
- Alphanumeric phases: A.1, A.2, B.1, C.1
- Numeric phases: 0.1, 0.2, 1.1, 2.1

### Adding verification Cards

Add a **verification Card** at the end of each Phase (before the GATE) that:
- Runs all tests for that phase
- Checks test coverage against the todolist
- Runs full regression

### Gate checks

Design gate_check.sh with:
- **Universal checks** (always include):
  - Unit tests pass
  - Full regression tests pass
  - SPEC-DECISION audit (grep for AI decisions)
- **Project-specific checks** (customize):
  - Backward compatibility verification
  - Contract/interface checks
  - Config consistency
  - Import validation

## Step 4: Generate All Files

### Directory Structure

```
Autodev/{project_name}/
├── autodev.sh              # Main pipeline script
├── system_prompt.md        # AI session prompt
├── gate_check.sh           # Automated gate checks
├── state                   # Progress tracker (empty file)
├── decisions.jsonl         # Decision audit trail (empty file, runtime populated)
├── logs/                   # Runtime logs (empty dir)
├── bug_reports/            # Bug Hunt reports per round (runtime populated)
└── cards/
    ├── {A.1}.md            # Task cards
    ├── {A.2}.md
    ├── ...
    └── phase_gate.md       # Phase gate audit template
```

`summary.md` is a runtime artifact. Do NOT pre-create it during scaffolding.

### File Templates

#### autodev.sh Template

```bash
#!/bin/bash
# ═══════════════════════════════════════════════════════════
#  {PROJECT_DISPLAY_NAME} — Automated Development Pipeline
#  用法: ./{AUTODEV_PATH}/autodev.sh [OPTIONS]
# ═══════════════════════════════════════════════════════════
set -euo pipefail

# ──── 路径配置 ────
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PROJECT_ROOT="$(cd "$SCRIPT_DIR/../.." && pwd)"
AUTODEV="$SCRIPT_DIR"
CARDS_DIR="$AUTODEV/cards"
LOGS_DIR="$AUTODEV/logs"
STATE_FILE="$AUTODEV/state"
SYSTEM_PROMPT="$AUTODEV/system_prompt.md"
GATE_SCRIPT="$AUTODEV/gate_check.sh"

# ──── macOS 兼容 ────
if command -v gtimeout &>/dev/null; then
    TIMEOUT_CMD="gtimeout"
elif command -v timeout &>/dev/null; then
    TIMEOUT_CMD="timeout"
else
    TIMEOUT_CMD=""
fi

# ──── 配置 ────
MODEL="${{ENV_PREFIX}_MODEL:-opus}"
VERIFY_MODEL="${{ENV_PREFIX}_VERIFY_MODEL:-opus}"   # 验收验证模型
CARD_TIMEOUT="${{ENV_PREFIX}_TIMEOUT:-900}"
GATE_MAX_RETRIES="${{ENV_PREFIX}_GATE_RETRIES:-10}"
TEST_MAX_RETRIES="${{ENV_PREFIX}_TEST_RETRIES:-10}"   # 测试自动修复最多重试次数
AC_MAX_RETRIES="${{ENV_PREFIX}_AC_RETRIES:-10}"      # 独立验收最多自修复轮次
DECISIONS_CONTEXT_LINES="${{ENV_PREFIX}_DECISIONS_CONTEXT_LINES:-120}"
DECISIONS_FILE="$AUTODEV/decisions.jsonl"            # 决策审计追踪
PHASE_BASELINE_FILE="$AUTODEV/.phase_baseline"       # 每个 Phase 的 diff 基线
SUMMARY_FILE="$AUTODEV/summary.md"                   # Pipeline 完成总结
BUG_HUNT_MAX_ROUNDS="${{ENV_PREFIX}_BUG_HUNT_ROUNDS:-15}"  # Bug Hunt 最大扫描轮次
BUG_REPORTS_DIR="$AUTODEV/bug_reports"                      # Bug 报告存放目录

# ──── 执行顺序 ────
ALL_STEPS=(
    {ALL_STEPS_CONTENT}
)

# ──── 颜色输出 ────
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'
BLUE='\033[0;34m'; CYAN='\033[0;36m'; BOLD='\033[1m'; NC='\033[0m'

log_info()  { echo -e "${BLUE}[INFO]${NC} $1"; }
log_ok()    { echo -e "${GREEN}[OK  ]${NC} $1"; }
log_warn()  { echo -e "${YELLOW}[WARN]${NC} $1"; }
log_fail()  { echo -e "${RED}[FAIL]${NC} $1"; }
log_title() {
    echo ""
    echo -e "${BOLD}${CYAN}=================================================${NC}"
    echo -e "${BOLD}${CYAN}  $1${NC}"
    echo -e "${BOLD}${CYAN}=================================================${NC}"
}

# ──── 安全 Prompt 传递 ────
# 所有动态内容（测试输出、AI 输出等）通过临时文件 + printf '%s\n' 传递给 claude，
# 避免: (1) 未引号 heredoc 导致 $、`、\ 被 shell 展开  (2) echo 超长字符串触发 ARG_MAX
# 模式: mktemp → { echo静态; printf '%s\n' "$动态变量"; cat <<'EOF' 静态 EOF } > tmpfile → claude < tmpfile

# ──── 进度管理 (100% 通用，不改) ────
is_completed() { grep -qxF "$1" "$STATE_FILE" 2>/dev/null; }
mark_completed() { echo "$1" >> "$STATE_FILE"; log_ok "完成: $1"; }
get_completed_count() {
    if [ -f "$STATE_FILE" ]; then wc -l < "$STATE_FILE" | tr -d ' '
    else echo "0"; fi
}

get_next_card_before_gate() {
    local gate_phase=$1 last_card=""
    for step in "${ALL_STEPS[@]}"; do
        local type="${step%%:*}" id="${step#*:}"
        if [ "$type" = "CARD" ]; then last_card="$id"
        elif [ "$type" = "GATE" ] && [ "$id" = "$gate_phase" ]; then break; fi
    done
    echo "${last_card:-{FIRST_CARD_ID}}"
}

# ──── Pipeline 文件防篡改保护 ────
# 防止 Claude agent 越界修改 state 文件或读取其他 Card 文件。
# 机制: chmod 444 (只读) + backup + 完整性校验 + 自动恢复。
_state_backup=""
protect_pipeline_files() {
    # 1. 备份 state 文件（不存在时跳过）
    [ ! -f "$STATE_FILE" ] && return 0
    _state_backup="${STATE_FILE}.bak.$$"
    cp "$STATE_FILE" "$_state_backup"
    # 2. state 文件设为只读（阻止 Claude Write 工具写入）
    chmod 444 "$STATE_FILE" 2>/dev/null || true
    # 3. 其他 Card 文件设为只读（阻止读取——Claude Read 工具不受影响，
    #    但防止 agent 修改其他 Card 内容）
    for f in "$CARDS_DIR"/*.md "$AUTODEV/autodev.sh" "$AUTODEV/system_prompt.md" "$AUTODEV/gate_check.sh"; do
        [ -f "$f" ] && chmod 444 "$f" 2>/dev/null || true
    done
}
unprotect_pipeline_files() {
    # 恢复 state 文件写权限
    chmod 644 "$STATE_FILE" 2>/dev/null || true
    # 完整性校验: state 文件是否被篡改
    if [ -n "$_state_backup" ] && [ -f "$_state_backup" ]; then
        if ! diff -q "$STATE_FILE" "$_state_backup" > /dev/null 2>&1; then
            log_fail "🚨 STATE FILE TAMPERED by Claude agent! Restoring from backup..."
            cp "$_state_backup" "$STATE_FILE"
            chmod 644 "$STATE_FILE"
        fi
        rm -f "$_state_backup"
    fi
    _state_backup=""
    # 恢复其他文件权限
    for f in "$CARDS_DIR"/*.md "$AUTODEV/autodev.sh" "$AUTODEV/system_prompt.md" "$AUTODEV/gate_check.sh"; do
        [ -f "$f" ] && chmod 644 "$f" 2>/dev/null || true
    done
}

# ──── 构建 Prompt ────
build_prompt() {
    local card_file="$CARDS_DIR/${1}.md"
    [ ! -f "$card_file" ] && { log_fail "Card 文件不存在: $card_file"; return 1; }
    {
        cat "$SYSTEM_PROMPT"; echo ""; echo "---"; echo ""
        echo "## 运行时信息"
        echo "DECISIONS_FILE: $DECISIONS_FILE"
        echo ""
        echo "## 当前进度"
        if [ -f "$STATE_FILE" ] && [ -s "$STATE_FILE" ]; then
            echo "已完成的 Card:"; cat "$STATE_FILE" | sed 's/^/- /'
        else echo "尚无已完成的 Card（首个任务）"; fi
        # 注入前序决策上下文
        if [ -f "$DECISIONS_FILE" ] && [ -s "$DECISIONS_FILE" ]; then
            echo ""; echo "## 前序决策记录（最近 ${DECISIONS_CONTEXT_LINES} 条）"
            echo '```jsonl'; tail -n "$DECISIONS_CONTEXT_LINES" "$DECISIONS_FILE"; echo '```'
        fi
        echo ""; echo "---"; echo ""; cat "$card_file"
    }
}

# ──── 执行单张 Card (通用骨架，TEST_CMD 可定制) ────
execute_card() {
    local card_id=$1 timestamp=$(date +%Y%m%d_%H%M%S)
    local log_file="$LOGS_DIR/card_${card_id//./_}_${timestamp}.log"
    log_title "Card $card_id"
    local prompt; prompt="$(build_prompt "$card_id")" || return 1
    log_info "Model: $MODEL | Timeout: ${CARD_TIMEOUT}s"
    log_info "Log: $log_file"; echo ""

    cd "$PROJECT_ROOT"
    local prompt_file; prompt_file=$(mktemp "${TMPDIR:-/tmp}/autodev_prompt.XXXXXX")
    printf '%s' "$prompt" > "$prompt_file"
    local exit_code=0
    protect_pipeline_files
    if [ -n "$TIMEOUT_CMD" ]; then
        $TIMEOUT_CMD "$CARD_TIMEOUT" claude -p --dangerously-skip-permissions --model "$MODEL" --verbose \
            < "$prompt_file" 2>&1 | tee "$log_file" || exit_code=$?
    else
        claude -p --dangerously-skip-permissions --model "$MODEL" --verbose \
            < "$prompt_file" 2>&1 | tee "$log_file" || exit_code=$?
    fi
    unprotect_pipeline_files
    rm -f "$prompt_file"

    [ $exit_code -eq 124 ] && { log_fail "Card $card_id 超时 (>${CARD_TIMEOUT}s)"; return 1; }
    [ $exit_code -ne 0 ] && { log_fail "Card $card_id 执行失败 (exit: $exit_code)"; return 1; }

    # 测试验证 + 自动修复循环
    echo ""
    local test_attempt=0 tests_passed=false
    local test_timeout=300  # 测试命令最多运行 5 分钟，防止交互式测试阻塞
    while [ $test_attempt -lt $TEST_MAX_RETRIES ]; do
        test_attempt=$((test_attempt + 1))
        log_info "运行测试 (第 ${test_attempt}/${TEST_MAX_RETRIES} 次)..."
        cd "$PROJECT_ROOT"
        local test_output test_exit=0
        if [ -n "$TIMEOUT_CMD" ]; then
            test_output=$($TIMEOUT_CMD "$test_timeout" {TEST_CMD} 2>&1) || test_exit=$?
        else
            test_output=$({TEST_CMD} 2>&1) || test_exit=$?
        fi
        if [ $test_exit -eq 124 ]; then
            log_fail "测试命令超时 (>${test_timeout}s)，可能存在交互式测试阻塞。请检查测试命令是否排除了需要交互输入的测试。"
            return 1
        fi
        [ $test_exit -eq 0 ] && tests_passed=true
        echo "$test_output" | tee -a "$log_file"
        [ "$tests_passed" = true ] && { log_ok "测试通过"; break; }

        if [ $test_attempt -lt $TEST_MAX_RETRIES ]; then
            log_warn "测试失败，AI 自动修复 ($test_attempt/$TEST_MAX_RETRIES)..."
            local fix_file; fix_file=$(mktemp "${TMPDIR:-/tmp}/autodev_fix.XXXXXX")
            {
                echo "Card $card_id 执行后测试失败，请修复。"
                echo ""
                echo "## 测试输出"
                echo '```'
                printf '%s\n' "$test_output"
                echo '```'
                echo ""
                cat <<'FIX_RULES_EOF'
## 规则
- 读取失败的测试文件和对应的实现代码
- 读取设计文档确认正确行为
- 修复后运行: {TEST_CMD}
- 只修复导致测试失败的问题，不要做额外改动
- 不能破坏现有测试（向后兼容）
FIX_RULES_EOF
            } > "$fix_file"
            cd "$PROJECT_ROOT"
            protect_pipeline_files
            claude -p --dangerously-skip-permissions --model "$MODEL" --verbose \
                < "$fix_file" 2>&1 | tee -a "$log_file" || true
            unprotect_pipeline_files
            rm -f "$fix_file"
        fi
    done
    [ "$tests_passed" != true ] && { log_fail "Card $card_id 测试修复失败"; return 1; }

    # ──── 独立验收验证 + 自动修复闭环 (Acceptance Verifier) ────
    local card_content
    card_content=$(cat "$CARDS_DIR/${card_id}.md")
    local ac_attempt=0 ac_passed=false
    while [ $ac_attempt -lt $AC_MAX_RETRIES ]; do
        ac_attempt=$((ac_attempt + 1))
        log_info "启动独立验收验证 (第 ${ac_attempt}/${AC_MAX_RETRIES} 次)..."
        local ac_file; ac_file=$(mktemp "${TMPDIR:-/tmp}/autodev_ac.XXXXXX")
        {
            echo "你是独立验收审计员（不是开发者）。验证 Card $card_id 的验收标准是否全部满足。"
            echo ""
            echo "## Card 完整内容"
            printf '%s\n' "$card_content"
            echo ""
            cat <<'VERIFY_STATIC_EOF'
## 请你：
1. 读取 Card 中"读取已有代码"列出的源文件，确认修改已就位
2. 运行 Card 中指定的测试命令，确认通过
3. 逐条检查验收标准（AC-1, AC-2, ...），判定 PASS 或 FAIL
4. 输出格式（严格遵守）：
   AC-1: PASS — 原因
   AC-2: FAIL — 原因
   ...
   VERDICT: ALL_PASS | HAS_FAILURES

重要：你是独立审计员。直接读文件和跑测试来验证，不要信任之前的 AI 输出。
VERIFY_STATIC_EOF
        } > "$ac_file"
        local verify_output verify_exit=0
        protect_pipeline_files
        verify_output=$(claude -p --dangerously-skip-permissions --model "$VERIFY_MODEL" --verbose < "$ac_file" 2>&1) || verify_exit=$?
        unprotect_pipeline_files
        rm -f "$ac_file"
        echo "$verify_output" | tee -a "$log_file"

        if [ $verify_exit -eq 0 ] && echo "$verify_output" | grep -q "VERDICT: ALL_PASS"; then
            log_ok "验收验证通过"
            ac_passed=true
            break
        fi

        if [ $verify_exit -ne 0 ]; then
            log_warn "验收验证 AI 异常退出 (exit: $verify_exit)，触发自恢复后重试"
        else
            log_warn "验收验证发现未满足的 AC，触发修复"
        fi

        local ac_fix_file; ac_fix_file=$(mktemp "${TMPDIR:-/tmp}/autodev_acfix.XXXXXX")
        {
            echo "验收审计发现以下 AC 未满足："
            echo ""
            printf '%s\n' "$verify_output"
            echo ""
            cat <<'AC_FIX_STATIC_EOF'
请修复未通过的验收标准。只修复未通过的项，不要改动已通过的部分。
修复后运行测试确认通过，并准备再次接受独立验收。
AC_FIX_STATIC_EOF
        } > "$ac_fix_file"
        protect_pipeline_files
        claude -p --dangerously-skip-permissions --model "$MODEL" --verbose \
            < "$ac_fix_file" 2>&1 | tee -a "$log_file"
        unprotect_pipeline_files
        rm -f "$ac_fix_file"

        # 修复后重跑测试；失败则继续下一轮自修复，不允许带病通过
        cd "$PROJECT_ROOT"
        local ac_test_exit=0
        if [ -n "$TIMEOUT_CMD" ]; then
            $TIMEOUT_CMD "$test_timeout" {TEST_CMD} 2>&1 | tee -a "$log_file" || ac_test_exit=$?
        else
            {TEST_CMD} 2>&1 | tee -a "$log_file" || ac_test_exit=$?
        fi
        if [ $ac_test_exit -eq 124 ]; then
            log_fail "验收修复后测试超时 (>${test_timeout}s)，疑似交互式测试阻塞"; return 1
        elif [ $ac_test_exit -ne 0 ]; then
            log_warn "Card $card_id 验收修复后测试仍失败，将继续自动修复"
        fi
    done
    [ "$ac_passed" != true ] && { log_fail "Card $card_id 验收验证未通过 (达到 AC 重试上限)"; return 1; }

    return 0
}

# ──── Phase Gate ────
run_phase_gate() {
    local phase=$1 timestamp=$(date +%Y%m%d_%H%M%S)
    local log_file="$LOGS_DIR/gate_${phase}_${timestamp}.log"
    log_title "Phase Gate: Phase $phase"

    # 1. gate_check.sh + 自动修复循环
    if [ -f "$GATE_SCRIPT" ]; then
        local attempt=0 gate_passed=false
        while [ $attempt -lt $GATE_MAX_RETRIES ]; do
            attempt=$((attempt + 1))
            log_info "门禁检查 (第 ${attempt}/${GATE_MAX_RETRIES} 次)..."
            cd "$PROJECT_ROOT"
            local gate_output
            gate_output=$(bash "$GATE_SCRIPT" 2>&1) && gate_passed=true
            echo "$gate_output" | tee -a "$log_file"
            [ "$gate_passed" = true ] && { log_ok "自动门禁通过"; break; }
            if [ $attempt -lt $GATE_MAX_RETRIES ]; then
                log_warn "门禁未通过，AI 自动修复..."
                local gate_fix_file; gate_fix_file=$(mktemp "${TMPDIR:-/tmp}/autodev_gatefix.XXXXXX")
                {
                    echo "门禁检查报告了以下问题，请修复:"
                    printf '%s\n' "$gate_output"
                    echo "读取设计文档和测试确认正确行为。只修复问题，不做额外改动。"
                } > "$gate_fix_file"
                protect_pipeline_files
                claude -p --dangerously-skip-permissions --model "$MODEL" --verbose \
                    < "$gate_fix_file" 2>&1 | tee -a "$log_file" || true
                unprotect_pipeline_files
                rm -f "$gate_fix_file"
            fi
        done
        [ "$gate_passed" != true ] && { log_fail "门禁经 $GATE_MAX_RETRIES 次修复仍未通过"; return 1; }
    fi

    # 2. AI 审计 (phase_gate.md)
    log_info "运行 AI 审计..."
    local gate_audit_file; gate_audit_file=$(mktemp "${TMPDIR:-/tmp}/autodev_audit.XXXXXX")
    {
        cat "$CARDS_DIR/phase_gate.md"
        echo ""
        echo "---"
        echo "当前审计: Phase $phase"
        echo "已完成的 Card:"
        grep "^CARD:" "$STATE_FILE" 2>/dev/null | sed 's/^CARD:/- /' || echo "（无）"
    } > "$gate_audit_file"
    protect_pipeline_files
    claude -p --dangerously-skip-permissions --model "$MODEL" --verbose \
        < "$gate_audit_file" 2>&1 | tee -a "$log_file" || true
    unprotect_pipeline_files
    rm -f "$gate_audit_file"
    log_ok "Phase Gate $phase 完成"

    # 3. 更新 Phase 基线（供下一阶段 phase-scoped diff）
    git rev-parse HEAD > "$PHASE_BASELINE_FILE" 2>/dev/null || true
    return 0
}

# ──── Pipeline 完成总结 ────
generate_summary() {
    log_title "生成 Pipeline 完成总结"
    local completed_cards
    completed_cards=$(grep "^CARD:" "$STATE_FILE" 2>/dev/null | sed 's/CARD://' | tr '\n' ', ' | sed 's/,$//')
    local completed_gates
    completed_gates=$(grep "^GATE:" "$STATE_FILE" 2>/dev/null | sed 's/GATE://' | tr '\n' ', ' | sed 's/,$//')

    # 对比 Pipeline 启动时的 baseline（含已提交 + 未提交的全部变更）
    local diff_ref="${PIPELINE_BASELINE:-HEAD}"
    local git_diff_stat git_diff_files
    cd "$PROJECT_ROOT"
    git_diff_stat=$(git diff --stat "$diff_ref" 2>/dev/null || echo "(no git changes)")
    # 已提交的变更 + 未提交的变更
    git_diff_files=$({ git diff --name-only "$diff_ref" 2>/dev/null; git diff --name-only 2>/dev/null; } | sort -u)
    local decisions_content=""
    if [ -f "$DECISIONS_FILE" ] && [ -s "$DECISIONS_FILE" ]; then
        decisions_content=$(cat "$DECISIONS_FILE")
    fi
    local log_files
    log_files=$(ls -1t "$LOGS_DIR"/*.log 2>/dev/null | head -20)

    local summary_file_prompt; summary_file_prompt=$(mktemp "${TMPDIR:-/tmp}/autodev_summary.XXXXXX")
    {
        cat <<'SUMMARY_STATIC_1'
你是 Pipeline 总结报告员。请根据以下信息生成一份结构化总结报告（Markdown 格式）。

## Pipeline 信息
SUMMARY_STATIC_1
        echo "- 项目: {PROJECT_DISPLAY_NAME}"
        echo "- 完成的 Cards: $completed_cards"
        echo "- 完成的 Gates: $completed_gates"
        echo ""
        echo "## Git 变更统计"
        echo '```'
        printf '%s\n' "$git_diff_stat"
        echo '```'
        echo ""
        echo "## 变更的文件列表"
        printf '%s\n' "$git_diff_files"
        echo ""
        echo "## 决策记录"
        echo '```jsonl'
        printf '%s\n' "$decisions_content"
        echo '```'
        echo ""
        cat <<'SUMMARY_STATIC_2'
## 请你：
1. 读取上述变更的文件，理解每个文件做了什么修改
2. 生成以下格式的总结报告，直接输出 Markdown 内容（不需要代码块包裹）：

# {PROJECT_DISPLAY_NAME} — Pipeline 完成总结

## 实现概要
（用 2-3 句话概括本次 pipeline 实现了什么）

## 变更清单
| 文件 | 变更类型 | 说明 |
|------|----------|------|
（每个变更文件一行：新增/修改/删除 + 一句话说明做了什么）

## 关键决策
（从 decisions.jsonl 提取，如无则写"无"）
- 【Card X.Y】决策描述 — 选择了方案 A，原因...

## 测试结果
（从 log 文件名推断哪些 Card 执行了，整体结论）

## 注意事项
（任何残余风险、TODO、或后续建议，如无则写"无"）

重要：只输出 Markdown 报告本身，不要用代码块包裹。
SUMMARY_STATIC_2
    } > "$summary_file_prompt"

    local summary_output
    summary_output=$(claude -p --dangerously-skip-permissions --model "$VERIFY_MODEL" --verbose < "$summary_file_prompt" 2>&1) || true
    rm -f "$summary_file_prompt"

    # 提取 Markdown 内容写入文件
    echo "$summary_output" > "$SUMMARY_FILE"
    log_ok "总结报告已生成: $SUMMARY_FILE"

    # 在终端直接展示总结
    echo ""
    echo -e "${BOLD}${CYAN}=================================================${NC}"
    echo -e "${BOLD}${CYAN}  Pipeline 完成总结${NC}"
    echo -e "${BOLD}${CYAN}=================================================${NC}"
    echo ""
    cat "$SUMMARY_FILE"
    echo ""
}

# ──── Bug Hunt Phase: 多轮 Bug 扫描 + 修复 + 回归测试 ────
run_bug_hunt() {
    log_title "Bug Hunt Phase 启动"
    local diff_ref="${PIPELINE_BASELINE:-HEAD}"
    local round=0

    while [ $round -lt $BUG_HUNT_MAX_ROUNDS ]; do
        round=$((round + 1))
        log_title "Bug Hunt — Round ${round}/${BUG_HUNT_MAX_ROUNDS}"

        # ──── [1] Bug Scan (独立审计员，只读) ────
        local report_file="$BUG_REPORTS_DIR/bug_report_round_${round}.md"
        local scan_file; scan_file=$(mktemp "${TMPDIR:-/tmp}/autodev_bughunt_scan.XXXXXX")
        cd "$PROJECT_ROOT"
        local git_diff_files
        git_diff_files=$({ git diff --name-only "$diff_ref" 2>/dev/null; git diff --name-only 2>/dev/null; } | sort -u)
        if [ -z "$git_diff_files" ]; then
            log_warn "无变更文件，跳过 Bug Hunt"
            break
        fi
        local summary_content=""
        [ -f "$SUMMARY_FILE" ] && summary_content=$(cat "$SUMMARY_FILE")
        local requirements_content=""
        for card_file in "$CARDS_DIR"/*.md; do
            [ -f "$card_file" ] && requirements_content="${requirements_content}
--- $(basename "$card_file") ---
$(cat "$card_file")
"
        done
        {
            cat <<'BH_SCAN_STATIC_1'
你是独立 Bug 审计员（不是开发者）。你的任务是对已完成的代码进行全面 Bug 扫描。

## 重要约束
- 你只有只读权限。不要修改、创建或删除任何文件。只读取和报告。
- 只报告真实存在的 bug，不要报告代码风格问题。
- 不要猜测或假设 bug，必须有源代码中的具体证据。

## 扫描方法
1. 读取下面列出的所有变更文件的源代码
2. 对照原始需求（Cards），逐一检查功能是否正确实现
3. 检查边界条件、错误处理、类型安全、安全漏洞
4. 检查现有测试的覆盖是否充分

## Bug 分级标准
- **P0**: 崩溃、数据丢失、安全漏洞、核心功能完全不可用
- **P1**: 功能未按需求实现、显著逻辑错误、重要边界条件缺失
- **P2**: 次要边界条件、缺少输入校验、错误信息不准确、测试覆盖不足

BH_SCAN_STATIC_1
            echo "## Pipeline 完成总结"
            printf '%s\n' "$summary_content"
            echo ""
            echo "## 变更的文件列表（请逐一读取源代码）"
            printf '%s\n' "$git_diff_files"
            echo ""
            echo "## 原始需求（Cards）"
            printf '%s\n' "$requirements_content"
            echo ""
            # 注入前几轮 bug 报告，防止重复报告已修复的 bug
            if [ $round -gt 1 ]; then
                echo "## 前几轮已发现并处理的 Bug（不要重复报告这些已修复的 bug）"
                for prev_rf in "$BUG_REPORTS_DIR"/bug_report_round_*.md; do
                    [ -f "$prev_rf" ] && { echo "--- $(basename "$prev_rf") ---"; cat "$prev_rf"; echo ""; }
                done
                echo ""
            fi
            cat <<'BH_SCAN_STATIC_2'
## 输出格式（严格遵守）

如果发现 bug，按以下格式输出：

### BUG-1 [P0] 简短描述
- 文件: path/to/file.py:行号
- 问题: 具体问题描述
- 预期: 正确的行为应该是什么
- 建议修复: 如何修复

### BUG-2 [P1] 简短描述
...

如果没有发现任何 P0/P1/P2 级别的 bug，输出：

VERDICT: NO_BUGS_FOUND

如果发现了 bug，在末尾输出：

VERDICT: BUGS_FOUND | P0:数量 P1:数量 P2:数量

重要：
- 你是独立审计员。直接读源文件来验证，不要信任之前的 AI 输出。
- 只报告有充分证据的真实 bug。
- 不要重复报告前几轮已发现并修复的 bug，只报告当前代码中仍然存在的新问题。
BH_SCAN_STATIC_2
        } > "$scan_file"

        log_info "Bug Scan 中 (VERIFY_MODEL: $VERIFY_MODEL)..."
        local scan_output scan_exit=0
        scan_output=$(claude -p --dangerously-skip-permissions --model "$VERIFY_MODEL" --verbose < "$scan_file" 2>&1) || scan_exit=$?
        rm -f "$scan_file"
        echo "$scan_output" > "$report_file"
        log_ok "Bug 报告已保存: $report_file"

        # ──── [2] 判断：有 P0/P1/P2？ ────
        if [ $scan_exit -eq 0 ] && echo "$scan_output" | grep -q "VERDICT: NO_BUGS_FOUND"; then
            log_ok "Bug Hunt 完成 — Round $round 未发现新 bug"
            break
        fi

        if [ $scan_exit -ne 0 ]; then
            log_warn "Bug Scan AI 异常退出 (exit: $scan_exit)，跳过本轮继续下一轮"
            continue
        fi

        log_warn "发现 bug，进入修复流程"

        # ──── [3]+[4]+[5] Bug Fix + Test + Verify 内循环 ────
        local fix_attempt=0 all_fixed=false
        while [ $fix_attempt -lt $AC_MAX_RETRIES ]; do
            fix_attempt=$((fix_attempt + 1))

            # ──── [3] Bug Fix (开发者 AI) ────
            log_info "Bug Fix (第 ${fix_attempt}/${AC_MAX_RETRIES} 次)..."
            local fix_file; fix_file=$(mktemp "${TMPDIR:-/tmp}/autodev_bughunt_fix.XXXXXX")
            {
                # 重试时注入上一轮 verify 反馈，帮助开发者理解哪些修复不充分
                if [ $fix_attempt -gt 1 ] && [ -n "${verify_output:-}" ]; then
                    echo "## 上一次修复的验证结果（请特别关注 NOT_FIXED 和 PARTIAL 的项目）"
                    printf '%s\n' "$verify_output"
                    echo ""
                fi
                echo "以下是独立审计员发现的 bug 报告："
                echo ""
                printf '%s\n' "$scan_output"
                echo ""
                cat <<'BH_FIX_STATIC_EOF'
## 你的任务
1. 读取报告中提到的每个文件
2. 修复所有列出的 P0/P1/P2 bug
3. 为每个 bug 编写对应的回归测试（确保 bug 不会再次出现）
4. 运行全量测试确认通过: {TEST_CMD}

## 规则
- 每个 bug 必须有对应的回归测试
- 回归测试放在项目现有的测试文件中（或与被修复代码相邻的测试文件中）
- 测试命名以 test_bug_ 或 test_regression_ 为前缀，便于识别
- 使用项目现有的测试框架和模式
- 不能破坏现有测试（向后兼容）
- 修复后运行测试确认全部通过
- 只修复报告中列出的 bug，不要做额外重构
BH_FIX_STATIC_EOF
            } > "$fix_file"
            cd "$PROJECT_ROOT"
            protect_pipeline_files
            claude -p --dangerously-skip-permissions --model "$MODEL" --verbose \
                < "$fix_file" 2>&1 | tee -a "$LOGS_DIR/bug_hunt_round_${round}.log" || true
            unprotect_pipeline_files
            rm -f "$fix_file"

            # ──── [4] 测试验证 (复用 TEST_MAX_RETRIES 机制) ────
            log_info "Bug Fix 后运行测试..."
            local test_attempt=0 tests_passed=false
            local test_timeout=300
            while [ $test_attempt -lt $TEST_MAX_RETRIES ]; do
                test_attempt=$((test_attempt + 1))
                log_info "运行测试 (第 ${test_attempt}/${TEST_MAX_RETRIES} 次)..."
                cd "$PROJECT_ROOT"
                local test_output test_exit=0
                if [ -n "$TIMEOUT_CMD" ]; then
                    test_output=$($TIMEOUT_CMD "$test_timeout" {TEST_CMD} 2>&1) || test_exit=$?
                else
                    test_output=$({TEST_CMD} 2>&1) || test_exit=$?
                fi
                if [ $test_exit -eq 124 ]; then
                    log_fail "测试超时 (>${test_timeout}s)"; break
                fi
                [ $test_exit -eq 0 ] && tests_passed=true
                echo "$test_output" | tee -a "$LOGS_DIR/bug_hunt_round_${round}.log"
                [ "$tests_passed" = true ] && { log_ok "测试通过"; break; }

                if [ $test_attempt -lt $TEST_MAX_RETRIES ]; then
                    log_warn "测试失败，AI 自动修复 ($test_attempt/$TEST_MAX_RETRIES)..."
                    local tfix_file; tfix_file=$(mktemp "${TMPDIR:-/tmp}/autodev_bh_tfix.XXXXXX")
                    {
                        echo "Bug Hunt 修复后测试失败，请修复。"
                        echo ""
                        echo "## 当前正在修复的 Bug 报告（参考上下文）"
                        printf '%s\n' "$scan_output"
                        echo ""
                        echo "## 测试输出"
                        echo '```'
                        printf '%s\n' "$test_output"
                        echo '```'
                        echo ""
                        cat <<'BH_TFIX_RULES_EOF'
## 规则
- 读取失败的测试文件和对应的实现代码
- 如果不确定测试的预期行为，读取上面的 bug 报告和需求文档来确认
- 优先修复实现代码而非修改测试的预期值（除非测试本身有错误）
- 修复后运行: {TEST_CMD}
- 只修复导致测试失败的问题，不要做额外改动
- 不能破坏现有测试
BH_TFIX_RULES_EOF
                    } > "$tfix_file"
                    cd "$PROJECT_ROOT"
                    protect_pipeline_files
                    claude -p --dangerously-skip-permissions --model "$MODEL" --verbose \
                        < "$tfix_file" 2>&1 | tee -a "$LOGS_DIR/bug_hunt_round_${round}.log" || true
                    unprotect_pipeline_files
                    rm -f "$tfix_file"
                fi
            done

            if [ "$tests_passed" != true ]; then
                log_warn "Bug Fix 后测试仍失败，继续下一轮修复尝试"
                continue
            fi

            # ──── [5] Fix Verify (独立审计员验证修复) ────
            log_info "验证 Bug 修复 (第 ${fix_attempt}/${AC_MAX_RETRIES} 次)..."
            local verify_file; verify_file=$(mktemp "${TMPDIR:-/tmp}/autodev_bughunt_verify.XXXXXX")
            {
                echo "你是独立审计员（只读角色，不要修改任何文件）。以下是之前发现的 bug 报告："
                echo ""
                printf '%s\n' "$scan_output"
                echo ""
                cat <<'BH_VERIFY_STATIC_EOF'
## 重要约束
- 你只有只读权限。不要修改、创建或删除任何文件。只读取和验证。

## 你的任务
1. 逐一检查报告中的每个 bug 是否已被修复（读取源文件确认）
2. 检查每个 bug 是否有对应的回归测试，且该测试确实能覆盖修复的场景（不是空壳测试）
3. 运行测试确认通过: {TEST_CMD}
4. 检查修复是否引入了新的问题

## 输出格式（严格遵守）
BUG-1: FIXED — 说明 | 回归测试: YES/NO
BUG-2: PARTIAL — 主要场景已修复但边界条件仍存在 | 回归测试: YES/NO
BUG-3: NOT_FIXED — 说明
...
VERDICT: ALL_FIXED | HAS_UNFIXED

说明：PARTIAL 视为未修复，计入 HAS_UNFIXED。

重要：直接读源文件和测试文件来验证，不要信任之前的 AI 输出。
BH_VERIFY_STATIC_EOF
            } > "$verify_file"
            local verify_output verify_exit=0
            protect_pipeline_files
            verify_output=$(claude -p --dangerously-skip-permissions --model "$VERIFY_MODEL" --verbose < "$verify_file" 2>&1) || verify_exit=$?
            unprotect_pipeline_files
            rm -f "$verify_file"
            echo "$verify_output" | tee -a "$LOGS_DIR/bug_hunt_round_${round}.log"

            if [ $verify_exit -eq 0 ] && echo "$verify_output" | grep -q "VERDICT: ALL_FIXED"; then
                log_ok "所有 bug 已修复并验证"
                all_fixed=true
                break
            fi

            log_warn "部分 bug 未修复，继续修复尝试"
        done

        if [ "$all_fixed" != true ]; then
            log_warn "Round $round: 经 $AC_MAX_RETRIES 次修复仍有未解决的 bug，进入下一轮全量扫描"
        fi
    done

    if [ $round -ge $BUG_HUNT_MAX_ROUNDS ]; then
        log_warn "Bug Hunt 达到最大轮次 ($BUG_HUNT_MAX_ROUNDS)，部分 bug 可能未解决"
        log_warn "请查看最新报告: $BUG_REPORTS_DIR/bug_report_round_${round}.md"
    fi

    # 更新 summary.md，追加 Bug Hunt 结果（只生成新章节，追加写入，防止丢失原始内容）
    log_info "更新 summary.md，追加 Bug Hunt 结果..."
    if [ ! -f "$SUMMARY_FILE" ]; then
        log_warn "summary.md 不存在，跳过 Bug Hunt 结果追加"
        return 0
    fi
    local bh_summary_file; bh_summary_file=$(mktemp "${TMPDIR:-/tmp}/autodev_bh_summary.XXXXXX")
    {
        echo "## Bug Hunt 报告数据"
        echo "总轮次: $round"
        for rf in "$BUG_REPORTS_DIR"/bug_report_round_*.md; do
            [ -f "$rf" ] && { echo "--- $(basename "$rf") ---"; cat "$rf"; echo ""; }
        done
        echo ""
        cat <<'BH_SUMMARY_STATIC_EOF'
请根据上面的 Bug Hunt 报告数据，只输出一个新的 Markdown 章节（不要包含已有的 summary 内容）。格式如下：

## Bug Hunt 结果
- 扫描轮次: N
- 发现并修复的 bug 数量: X
- 剩余未修复: Y（如有）
- 新增回归测试: Z 个

### 修复的 Bug 列表
| Bug ID | 优先级 | 描述 | 状态 |
|--------|--------|------|------|

重要：只输出上面这个章节的 Markdown 内容。不要输出其他内容，不要用代码块包裹。
BH_SUMMARY_STATIC_EOF
    } > "$bh_summary_file"
    local bh_section
    bh_section=$(claude -p --dangerously-skip-permissions --model "$VERIFY_MODEL" --verbose < "$bh_summary_file" 2>>"$LOGS_DIR/bug_hunt_summary.log") || true
    rm -f "$bh_summary_file"
    { echo ""; echo "$bh_section"; } >> "$SUMMARY_FILE"
    log_ok "summary.md 已更新（追加 Bug Hunt 结果）"
}

# ──── CLI ────
show_help() {
    echo "{PROJECT_DISPLAY_NAME} — Automated Development Pipeline"
    echo ""
    echo "用法: ./{AUTODEV_PATH}/autodev.sh [OPTIONS]"
    echo ""
    echo "选项:"
    echo "  --from CARD_ID    从指定 Card 开始 (如 --from B.1)"
    echo "  --model MODEL     Claude 模型 (默认: opus)"
    echo "  --reset           清除所有进度，从头开始"
    echo "  --dry-run         只显示执行计划，不实际执行"
    echo "  --status          显示当前进度"
    echo "  --help            显示此帮助"
}

show_status() {
    local total=${#ALL_STEPS[@]} done=$(get_completed_count)
    echo ""; echo "{PROJECT_DISPLAY_NAME} 开发进度: $done / $total"; echo ""
    for step in "${ALL_STEPS[@]}"; do
        local id="${step#*:}" type="${step%%:*}"
        if is_completed "$step"; then
            echo -e "  ${GREEN}[DONE]${NC} [$type] $id"
        else
            echo -e "  ⬜ [$type] $id"
        fi
    done
}

main() {
    local start_from="" dry_run=false
    while [[ $# -gt 0 ]]; do
        case $1 in
            --from)     start_from="$2"; shift 2 ;;
            --model)    MODEL="$2"; shift 2 ;;
            --reset)    rm -f "$STATE_FILE"; log_info "进度已清除"; shift ;;
            --dry-run)  dry_run=true; shift ;;
            --status)   show_status; exit 0 ;;
            --help)     show_help; exit 0 ;;
            *)          log_fail "未知选项: $1"; show_help; exit 1 ;;
        esac
    done

    mkdir -p "$LOGS_DIR"
    mkdir -p "$BUG_REPORTS_DIR"
    touch "$STATE_FILE"

    # 异常退出时恢复文件权限（防止 chmod 444 残留）
    trap 'unprotect_pipeline_files 2>/dev/null; rm -f "${STATE_FILE}.bak.$$" 2>/dev/null' EXIT INT TERM

    # 记录 Pipeline 起点（用于最终 summary diff）
    PIPELINE_BASELINE=$(cd "$PROJECT_ROOT" && git rev-parse HEAD 2>/dev/null || echo "")

    log_title "{PROJECT_DISPLAY_NAME} — Pipeline 启动"
    log_info "Model: $MODEL | Progress: $(get_completed_count)/${#ALL_STEPS[@]}"

    if [ "$dry_run" = true ]; then
        for step in "${ALL_STEPS[@]}"; do
            local type="${step%%:*}" id="${step#*:}"
            if is_completed "$step"; then
                echo -e "  ${GREEN}[SKIP]${NC} $type $id"
            else
                echo -e "  ${YELLOW}[TODO]${NC} $type $id"
            fi
        done
        exit 0
    fi

    local skip=false cards_executed=0 start_time=$(date +%s)
    [ -n "$start_from" ] && skip=true

    for step in "${ALL_STEPS[@]}"; do
        local type="${step%%:*}" id="${step#*:}"
        if [ "$skip" = true ]; then
            [ "$id" = "$start_from" ] && skip=false || continue
        fi
        is_completed "$step" && { log_info "跳过已完成: $type $id"; continue; }

        if [ "$type" = "GATE" ]; then
            run_phase_gate "$id" && mark_completed "$step" || { log_fail "Gate $id 失败"; exit 1; }
        elif [ "$type" = "CARD" ]; then
            execute_card "$id" && { mark_completed "$step"; cards_executed=$((cards_executed + 1)); } \
                || { log_fail "Card $id 失败，重跑: ./{AUTODEV_PATH}/autodev.sh --from $id"; exit 1; }
        fi
    done

    generate_summary
    run_bug_hunt
    local elapsed=$(( ($(date +%s) - start_time) / 60 ))
    log_title "Pipeline 完成！Cards: $cards_executed | Bug Hunt: max ${BUG_HUNT_MAX_ROUNDS} rounds | 耗时: ${elapsed}m"
}
main "$@"
```

**IMPORTANT**: Do NOT use this template literally. Generate the FULL script with all functions expanded (like the real autodev.sh files). The template above shows the *structure* — the actual output must be a complete, runnable bash script.

#### system_prompt.md Template

Structure (customize content):

```markdown
# {PROJECT_DISPLAY_NAME} — 自动开发会话

## 你的角色
你是 {PROJECT_NAME} 的开发者。严格按照设计文档实现，不添加文档未要求的功能。

## 项目文件
| 文件 | 路径 | 用途 |
|------|------|------|
| **设计文档** | `{SPEC_PATH}` | 唯一设计真相源 |
| **任务清单** | `{TODOLIST_PATH}` | 详细任务分解 |
{ADDITIONAL_FILES}

## TDD 流程
Step 1: RED → Step 2: GREEN → Step 3: SPEC → Step 4: LINT → Step 5: RUN

## 核心约束（从 todolist 的 P1/P2 修正中提取）
{PROJECT_CONSTRAINTS}

## 决策协议（无人值守环境）

本会话无人类在场。所有需要"人类确认"的场景，由 **AI 互相确认**替代。

### Level 1: SPEC-DECISION（自决 — 低风险歧义）

适用：参数命名、代码风格、小范围实现选择等不影响架构的决策。

```
Round 1: 列举所有可能方案
Round 2: 推演每个方案的影响 + 与文档其他部分的一致性
Round 3: 选择最优方案 + 标注残余风险
```

在代码中标注: `# SPEC-DECISION: chose B over A because ...`

### Level 2: AI-REVIEW（互审 — 高风险决策）

适用：架构变更、P1 约束相关、向后兼容影响、跨文件接口修改、设计文档歧义有多种合理解读。

**触发条件**（满足任一即触发 AI-REVIEW 而非 SPEC-DECISION）：
- 涉及 P1/P2 约束的实现选择
- 影响 2+ 个文件的接口变更
- SPEC-DECISION Round 3 仍有"高"残余风险
- 修改可能破坏现有测试或向后兼容性

**Review Agent 专业化**：根据触发原因自动选择审计角色：

| 触发原因 | 审计角色 | 审查重点 |
|---------|---------|---------|
| 向后兼容影响 | **Compatibility Reviewer** | 现有调用方是否不受影响、默认值是否保持、回归测试覆盖 |
| 跨文件接口变更 | **Interface Reviewer** | 签名一致性、类型匹配、调用链完整性 |
| P1/P2 约束相关 | **Constraint Reviewer** | 约束是否被满足、边界条件、降级路径 |
| 设计文档歧义 | **Spec Reviewer** | 文档各章节一致性、与已实现代码的匹配度 |

**执行方式**：使用 `Task` tool 启动一个独立的 review agent：

```
Task(subagent_type="general-purpose", prompt="""
你是 {PROJECT_NAME} 的 {REVIEWER_ROLE}。请审查以下决策：

## 决策上下文
{描述当前面临的选择}

## 候选方案
A: {方案A描述}
B: {方案B描述}

## 相关约束
{从 todolist/spec 中提取的 P1/P2 约束}

## 请你：
1. 读取 `{SPEC_PATH}` 相关章节验证两个方案的合规性
2. 读取 `{TODOLIST_PATH}` 确认任务意图
3. 读取涉及的源文件评估影响范围
4. 给出你的独立判断：选择哪个方案 + 理由
5. 如果两个方案都不合适，提出你的方案
6. 对每个发现标注严重级别：BLOCK / WARN / SUGGEST

严重级别定义：
- BLOCK: 必须修改才能继续，违反 P1 约束或破坏现有行为
- WARN: 建议修改，存在风险但不致命，可标注后跳过
- SUGGEST: 改进建议，记录但不阻塞流程
""")
```

**结果处理**：
- 两个 AI 一致 → 采用共识方案，标注 `# AI-REVIEW: consensus on B`
- 两个 AI 分歧 → 按严重级别处理：
  - 分歧项为 BLOCK → 采用**更保守**的方案，标注 `# AI-REVIEW: disagreement/BLOCK, chose conservative A`
  - 分歧项为 WARN → 采用 Card AI 的方案，标注 `# AI-REVIEW: disagreement/WARN, proceeded with B`
  - 分歧项为 SUGGEST → 记录但不改变方案

**决策记录**：每次 AI-REVIEW 完成后，追加到 `decisions.jsonl`（见决策审计追踪）。

### Level 3: AI-GATE（阻断 — 关键节点强制审计）

适用：Phase Gate、Card 完成前的最终验证。已内置于 autodev.sh 的 Phase Gate 流程中。

**与 Phase Gate 的关系**：AI-GATE 是 Phase Gate 的子集，由 gate_check.sh + AI 审计会话自动执行，无需 Card 内部触发。

### 决策审计追踪

每次 SPEC-DECISION 或 AI-REVIEW 完成后，**必须**追加一条记录。

路径来源：`build_prompt()` 会注入 `DECISIONS_FILE: /path/to/decisions.jsonl`，Card AI 从中获取路径。

```python
import json, datetime

# DECISIONS_FILE 路径从 prompt 的"运行时信息"部分获取
decisions_path = "Autodev/{project_name}/decisions.jsonl"  # 示例

decision = {
    "timestamp": datetime.datetime.now(datetime.timezone.utc).isoformat(),
    "card": "B.1",                            # 当前 Card ID
    "level": "AI-REVIEW",                     # SPEC-DECISION | AI-REVIEW
    "reviewer_role": "Compatibility Reviewer", # AI-REVIEW 填写, SPEC-DECISION 为 null
    "trigger": "backward compat impact",
    "options": ["A: keyword-only", "B: positional"],
    "chosen": "A",
    "severity": "BLOCK",                      # BLOCK | WARN | SUGGEST
    "consensus": True,                        # AI-REVIEW: 两个 AI 是否一致
    "rationale": "preserves existing callers",
    "residual_risk": "none",
    "file": "src/engine.py",
    "line": 42
}

with open(decisions_path, "a") as f:
    f.write(json.dumps(decision, ensure_ascii=False) + "\n")
```

**用途**：
- Gate 审计时统计决策分布（BLOCK/WARN/SUGGEST 数量）
- 检测应走 AI-REVIEW 但只走了 SPEC-DECISION 的遗漏
- `build_prompt()` 自动将已有记录注入后续 Card 的 prompt，提供前序决策上下文

## 禁止事项（安全边界 — 违反将导致 Pipeline 回滚）

### 范围边界（最高优先级）
- ❌ **严禁实现当前 Card 以外的任何功能** — 你只负责当前 Card 的验收标准，不要"顺便"做其他 Card 的工作
- ❌ **严禁读取 cards/ 目录下其他 Card 文件** — 你不需要知道后续 Card 的内容，也不应提前实现
- ❌ **严禁修改 Autodev/ 目录下的任何文件** — 包括 state, decisions.jsonl 以外的文件、autodev.sh、system_prompt.md、gate_check.sh、cards/*.md（state 和 autodev.sh 在执行期间为只读，写入会报错）
- ❌ **严禁写入 state 文件** — 进度管理完全由 autodev.sh 控制，不是你的职责

### 实现约束
- ❌ 添加设计文档未描述的功能
- ❌ 跳过测试
- ❌ 修改其他 Card 已实现的代码（除非当前 Card 明确要求）

### decisions.jsonl 例外
- ✅ 你**可以且应该** append 到 decisions.jsonl（记录 SPEC-DECISION / AI-REVIEW）
- ❌ 但**不可**删除或覆盖 decisions.jsonl 的已有内容
{PROJECT_SPECIFIC_PROHIBITIONS}

## Skill 使用规则

⚠️ **全自动流水线**: 本会话无人类在场，需要人类确认的场景由 AI 互审替代。

### 可用 Skill（已验证自动化安全）

| 时机 | Skill | 说明 |
|------|-------|------|
| **开始写代码前** | `/test-driven-development` | 启动 TDD 流程，先写测试再实现 |
| **测试失败时** | `/systematic-debugging` | 系统化排查，禁止盲猜修复 |
| **Card 完成前** | `/verification-before-completion` | 运行验证命令确认全部通过 |
| **Card 含 2+ 独立文件时** | `/dispatching-parallel-agents` | 并行开发独立模块 |

### 原需人类确认的 Skill → AI 互审替代

| 原 Skill | 原阻塞原因 | 替代方案 |
|----------|-----------|----------|
| `/brainstorming` | 需用户逐节批准设计 | **AI-REVIEW**: spawn review agent 审核设计决策 |
| `/subagent-driven-development` | 子代理提问需人类回答 | **AI-REVIEW**: 决策点由 review agent 确认，执行用 `/dispatching-parallel-agents` |
| `/requesting-code-review` | Critical issue 需人类判断 | **AI-REVIEW**: spawn review agent 做独立 code review，Phase Gate 做最终审计 |

### Skill 调用规则

1. **TDD 必调**: 每张 Card 开始实现时，必须先调用 `/test-driven-development`
2. **调试必调**: 测试失败 ≥1 次后，必须调用 `/systematic-debugging`
3. **完成必调**: 声明 Card 完成前，必须调用 `/verification-before-completion`
4. **并行优先**: 当 Card 包含多个独立文件时，优先用 `/dispatching-parallel-agents`
5. **决策分级**: 低风险用 SPEC-DECISION 自决，高风险用 AI-REVIEW 互审
```

#### Card Template

```markdown
# Card {CARD_ID}: {CARD_TITLE}

## 读取设计文档
从 `{SPEC_PATH}` 读取：
- {RELEVANT_SECTIONS}

从 `{TODOLIST_PATH}` 读取：
- {RELEVANT_TASKS}

## 读取已有代码
- {EXISTING_FILES_TO_READ}

## 任务
{TASKS_WITH_CODE_SNIPPETS}

## 验收标准
- [ ] AC-1: {ACCEPTANCE_CRITERION_1}
- [ ] AC-2: {ACCEPTANCE_CRITERION_2}
...
- [ ] AC-N: 所有测试通过
- [ ] AC-N+1: 现有测试不受影响（如适用）
```

#### gate_check.sh Template

```bash
#!/bin/bash
set -e
PROJECT_ROOT="$(cd "$(dirname "$0")/../.." && pwd)"
AUTODEV_DIR="$(cd "$(dirname "$0")" && pwd)"
cd "$PROJECT_ROOT"
PASS=0; FAIL=0; WARN=0
check_pass() { echo -e "\033[0;32m  ✅ $1\033[0m"; PASS=$((PASS+1)); }
check_fail() { echo -e "\033[0;31m  ❌ $1\033[0m"; FAIL=$((FAIL+1)); }
check_warn() { echo -e "\033[1;33m  ⚠️  $1\033[0m"; WARN=$((WARN+1)); }

echo "═══════════════════════════════════════════════"
echo "  {PROJECT_DISPLAY_NAME} — Phase Gate Checks"
echo "═══════════════════════════════════════════════"

# ──── 通用检查 ────

# 1. 核心单元测试
# 2. 全量回归测试

# 3. SPEC-DECISION / AI-REVIEW 审计
echo ""
echo "── 决策审计 ──"
spec_count=$(grep -r "SPEC-DECISION" {SOURCE_DIRS} 2>/dev/null | wc -l | tr -d ' ')
review_count=$(grep -r "AI-REVIEW" {SOURCE_DIRS} 2>/dev/null | wc -l | tr -d ' ')
echo "  SPEC-DECISION 标注: $spec_count 处"
echo "  AI-REVIEW 标注: $review_count 处"

# 4. AI-REVIEW 覆盖率检查
# 检查跨文件变更是否有对应的 AI-REVIEW 记录
if [ -f "$AUTODEV_DIR/decisions.jsonl" ] && [ -s "$AUTODEV_DIR/decisions.jsonl" ]; then
    block_count=$(grep -c '"severity": *"BLOCK"' "$AUTODEV_DIR/decisions.jsonl" 2>/dev/null || echo 0)
    warn_count=$(grep -c '"severity": *"WARN"' "$AUTODEV_DIR/decisions.jsonl" 2>/dev/null || echo 0)
    total_decisions=$(wc -l < "$AUTODEV_DIR/decisions.jsonl" | tr -d ' ')
    review_decisions=$(grep -c '"level": *"AI-REVIEW"' "$AUTODEV_DIR/decisions.jsonl" 2>/dev/null || echo 0)
    echo "  decisions.jsonl: $total_decisions 条记录 (AI-REVIEW: $review_decisions, BLOCK: $block_count, WARN: $warn_count)"
    # 检查是否有未解决的 BLOCK
    unresolved_blocks=$(grep '"severity": *"BLOCK"' "$AUTODEV_DIR/decisions.jsonl" | grep '"consensus": *false' | wc -l | tr -d ' ')
    if [ "$unresolved_blocks" -gt 0 ]; then
        check_fail "存在 $unresolved_blocks 个未达成共识的 BLOCK 级决策"
    else
        check_pass "所有 BLOCK 级决策已达成共识"
    fi
else
    check_warn "decisions.jsonl 为空或不存在，无法审计决策覆盖率"
fi

# 5. 跨文件变更 vs AI-REVIEW 匹配检查（Phase 级）
phase_base_file="$AUTODEV_DIR/.phase_baseline"
phase_base_ref=""
[ -f "$phase_base_file" ] && phase_base_ref=$(cat "$phase_base_file")
if [ -n "$phase_base_ref" ] && git rev-parse --verify "${phase_base_ref}^{commit}" >/dev/null 2>&1; then
    changed_files=$(git diff --name-only "$phase_base_ref" -- 2>/dev/null | wc -l | tr -d ' ')
    echo "  Phase 基线: $phase_base_ref"
    phase_review_count=0
    while IFS= read -r changed_file; do
        [ -f "$changed_file" ] || continue
        file_review_count=$(grep -c "AI-REVIEW" "$changed_file" 2>/dev/null || true)
        phase_review_count=$((phase_review_count + file_review_count))
    done < <(git diff --name-only "$phase_base_ref" -- 2>/dev/null)
else
    changed_files=$(git diff --name-only HEAD 2>/dev/null | wc -l | tr -d ' ')
    phase_review_count=$review_count
    check_warn "未找到有效 .phase_baseline，暂回退到工作区 diff；建议每个 Gate 成功后更新该基线"
fi
if [ "$changed_files" -gt 3 ] && [ "$phase_review_count" -eq 0 ]; then
    check_warn "本 Phase 变更了 $changed_files 个文件但代码中无 AI-REVIEW 标注"
else
    check_pass "AI-REVIEW 覆盖率正常 ($changed_files 文件变更, 本 Phase $phase_review_count 处 AI-REVIEW 标注)"
fi

# ──── 项目特定检查 (定制) ────
# 6-N. {PROJECT_SPECIFIC_CHECKS}

# ──── 汇总 ────
echo ""
echo "═══════════════════════════════════════════════"
echo "结果: ✅ $PASS | ❌ $FAIL | ⚠️  $WARN"
[ $FAIL -gt 0 ] && exit 1 || exit 0
```

#### phase_gate.md Template

```markdown
# Phase Gate 审计

## 你的角色
你是 {PROJECT_NAME} 的合规审计员。

## 审计步骤
1. 读取设计文档
2. 读取代码变更（git diff）
3. 逐项检查（项目特定检查清单）
4. 运行测试
5. **决策审计**：读取 `{AUTODEV_DIR}/decisions.jsonl`，检查：
   - 所有 BLOCK 级决策是否已达成共识
   - 跨文件变更是否有对应 AI-REVIEW 记录
   - SPEC-DECISION 的残余风险是否合理
   - 统计本 Phase 决策分布（SPEC-DECISION vs AI-REVIEW, BLOCK/WARN/SUGGEST）
6. 输出审计报告（通过项 / P0-P2 问题 / 决策审计摘要 / 结论）
```

## Step 5: Verify

After generating all files:

1. `chmod +x autodev.sh gate_check.sh`
2. Run `./autodev.sh --dry-run` — verify all steps listed correctly
3. Run `./autodev.sh --status` — verify display works
4. Run `./autodev.sh --help` — verify help text
5. Verify all card files exist: `ls cards/`
6. Verify state file is empty: `cat state`
7. Verify decisions.jsonl exists and is empty: `test -f decisions.jsonl && [ ! -s decisions.jsonl ]`
8. Verify summary.md does NOT exist yet (it's auto-generated at runtime)
9. Verify generated files contain no unresolved placeholders/stubs:
   - `! rg -n '\{(PROJECT|SPEC|TODOLIST|TEST_CMD|ALL_STEPS_CONTENT|FIRST_CARD_ID|SOURCE_DIRS|AUTODEV_PATH|ENV_PREFIX|ADDITIONAL_FILES|RELEVANT|EXISTING_FILES|ACCEPTANCE_CRITERION|CARD_TITLE|REVIEWER_ROLE|TASKS_WITH)[^}]*\}' autodev.sh gate_check.sh system_prompt.md cards/*.md`
   - `! rg -n 'show_help\(\) \{ \.\.\. \}|show_status\(\) \{ \.\.\. \}' autodev.sh`

## Key Principles

1. **Todolist is the source of truth** — every Card traces back to todolist tasks
2. **P1/P2 fixes become constraints** — embed audit corrections into Card acceptance criteria
3. **Test command is sacred** — must match what the todolist specifies
4. **Test command must be non-interactive** — the generated test command must run fully unattended with no stdin prompts. Scan the test directory for interactive tests (input(), getpass(), wallet prompts, manual e2e) and add --ignore flags to exclude them. A test command that hangs waiting for input will block the entire pipeline.
5. **Backward compatibility** — if todolist mentions fallback/compatibility, enforce in gate checks
6. **No gold-plating** — Cards only implement what the todolist describes
7. **Each Card is self-contained** — reads its own context, doesn't assume prior Card output was read

## Generator Self-Check

After generating all files, verify these cross-cutting concerns:

1. **system_prompt.md** contains the full 3-level decision protocol (copy from template verbatim)
2. **Reviewer role table** is customized for the project (add/remove roles as needed; severity levels BLOCK/WARN/SUGGEST are universal — never modify)
3. **gate_check.sh** includes decisions.jsonl audit checks
4. **phase_gate.md** includes decision audit as step 5
5. **All fix prompts** include: test output verbatim, spec/todolist references, "no extra changes", backward compatibility reminder
6. **`decisions.jsonl` path** flows correctly: config var → `build_prompt()` injects `DECISIONS_FILE:` → Card AI reads it from prompt
