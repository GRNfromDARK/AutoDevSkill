# Bug Hunt Phase Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a multi-round Bug Hunt phase to the autodev pipeline template that scans for P0/P1/P2 bugs after development, fixes them with regression tests, and loops until clean.

**Architecture:** All changes are in `skills/auto-dev/SKILL.md` (the template) and `README.md`. The Bug Hunt phase is a new function `run_bug_hunt()` called after `generate_summary()` in `main()`. It uses two AI roles: Bug Hunter (VERIFY_MODEL, read-only scan) and Bug Fixer (MODEL, fix code + write tests). The inner fix-verify loop reuses `AC_MAX_RETRIES`; the outer scan loop uses new `BUG_HUNT_MAX_ROUNDS`.

**Tech Stack:** Bash template in SKILL.md, Markdown

---

### Task 1: Add Bug Hunt config variables

**Files:**
- Modify: `skills/auto-dev/SKILL.md:197-207` (config section)

**Step 1: Add config variables after line 207**

Add after `SUMMARY_FILE` line:

```bash
BUG_HUNT_MAX_ROUNDS="${{ENV_PREFIX}_BUG_HUNT_ROUNDS:-15}"  # Bug Hunt 最大扫描轮次
BUG_REPORTS_DIR="$AUTODEV/bug_reports"                      # Bug 报告存放目录
```

**Step 2: Add mkdir in main() setup**

In `main()` around line 665, after `mkdir -p "$LOGS_DIR"`, add:

```bash
mkdir -p "$BUG_REPORTS_DIR"
```

**Step 3: Commit**

```bash
git add skills/auto-dev/SKILL.md
git commit -m "feat: add Bug Hunt config variables"
```

---

### Task 2: Add run_bug_hunt() function

**Files:**
- Modify: `skills/auto-dev/SKILL.md` — insert new function between `generate_summary()` (ends ~line 621) and `# ──── CLI ────` (line 623)

**Step 1: Add the run_bug_hunt() function**

Insert after line 621 (`}` closing generate_summary), before `# ──── CLI ────`:

```bash
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

## 扫描方法
1. 读取下面列出的所有变更文件的源代码
2. 对照原始需求（Cards），逐一检查功能是否正确实现
3. 检查边界条件、错误处理、类型安全、安全漏洞
4. 检查现有测试的覆盖是否充分
5. 只报告真实存在的 bug，不要报告代码风格问题

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

重要：你是独立审计员。直接读源文件来验证，不要信任之前的 AI 输出。只报告有充分证据的真实 bug。
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
                        echo "## 测试输出"
                        echo '```'
                        printf '%s\n' "$test_output"
                        echo '```'
                        echo ""
                        cat <<'BH_TFIX_RULES_EOF'
## 规则
- 读取失败的测试文件和对应的实现代码
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
                echo "你是独立审计员。以下是之前发现的 bug 报告："
                echo ""
                printf '%s\n' "$scan_output"
                echo ""
                cat <<'BH_VERIFY_STATIC_EOF'
## 你的任务
1. 逐一检查报告中的每个 bug 是否已被修复
2. 检查每个 bug 是否有对应的回归测试
3. 运行测试确认通过: {TEST_CMD}
4. 检查修复是否引入了新的问题

## 输出格式（严格遵守）
BUG-1: FIXED — 说明 | 回归测试: YES/NO
BUG-2: FIXED — 说明 | 回归测试: YES/NO
BUG-3: NOT_FIXED — 说明
...
VERDICT: ALL_FIXED | HAS_UNFIXED

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

    # 更新 summary.md，追加 Bug Hunt 结果
    log_info "更新 summary.md，追加 Bug Hunt 结果..."
    local bh_summary_file; bh_summary_file=$(mktemp "${TMPDIR:-/tmp}/autodev_bh_summary.XXXXXX")
    {
        echo "请在以下 summary.md 末尾追加一个 '## Bug Hunt 结果' 章节。"
        echo ""
        echo "## 当前 summary.md"
        cat "$SUMMARY_FILE"
        echo ""
        echo "## Bug Hunt 报告"
        echo "总轮次: $round"
        for rf in "$BUG_REPORTS_DIR"/bug_report_round_*.md; do
            [ -f "$rf" ] && { echo "--- $(basename "$rf") ---"; cat "$rf"; echo ""; }
        done
        echo ""
        cat <<'BH_SUMMARY_STATIC_EOF'
请追加以下格式到 summary.md 末尾：

## Bug Hunt 结果
- 扫描轮次: N
- 发现并修复的 bug 数量: X
- 剩余未修复: Y（如有）
- 新增回归测试: Z 个

### 修复的 Bug 列表
| Bug ID | 优先级 | 描述 | 状态 |
|--------|--------|------|------|

直接输出完整的更新后的 summary.md（不需要代码块包裹）。
BH_SUMMARY_STATIC_EOF
    } > "$bh_summary_file"
    local updated_summary
    updated_summary=$(claude -p --dangerously-skip-permissions --model "$VERIFY_MODEL" --verbose < "$bh_summary_file" 2>&1) || true
    rm -f "$bh_summary_file"
    echo "$updated_summary" > "$SUMMARY_FILE"
    log_ok "summary.md 已更新（含 Bug Hunt 结果）"
}
```

**Step 2: Commit**

```bash
git add skills/auto-dev/SKILL.md
git commit -m "feat: add run_bug_hunt() function template"
```

---

### Task 3: Integrate Bug Hunt into main()

**Files:**
- Modify: `skills/auto-dev/SKILL.md` — main() function, around line 708

**Step 1: Add run_bug_hunt() call after generate_summary**

Change:

```bash
    generate_summary
    log_title "Pipeline 完成！Cards: $cards_executed | 耗时: ${elapsed}m"
```

To:

```bash
    generate_summary
    run_bug_hunt
    local elapsed=$(( ($(date +%s) - start_time) / 60 ))
    log_title "Pipeline 完成！Cards: $cards_executed | Bug Hunt: ${BUG_HUNT_MAX_ROUNDS} rounds max | 耗时: ${elapsed}m"
```

Note: move the elapsed calculation after bug_hunt since bug hunt adds significant time.

**Step 2: Commit**

```bash
git add skills/auto-dev/SKILL.md
git commit -m "feat: integrate Bug Hunt into main pipeline flow"
```

---

### Task 4: Update SKILL.md description sections

**Files:**
- Modify: `skills/auto-dev/SKILL.md:14-27` (What is Autodev section)

**Step 1: Add Bug Hunt to the feature list**

Add item 10 after item 9:

```markdown
10. **Bug Hunt phase** — multi-round bug scanning + fixing after development: independent auditor finds P0/P1/P2 bugs, developer AI fixes with regression tests, loops until clean (up to 15 rounds)
```

**Step 2: Update Output section**

Add `bug_reports/` to the directory structure (around line 155-162).

**Step 3: Commit**

```bash
git add skills/auto-dev/SKILL.md
git commit -m "docs: update SKILL.md description with Bug Hunt phase"
```

---

### Task 5: Update README.md

**Files:**
- Modify: `README.md`

**Step 1: Add Bug Hunt to Key Features (after line 58)**

```markdown
- **Bug Hunt phase** — multi-round P0/P1/P2 bug scanning after development, with auto-fix + regression tests
```

**Step 2: Update pipeline positioning diagram (line 43-44)**

```
auto-requirement → auto-todo → auto-dev → [Bug Hunt]
(产品决策层)         (工程任务层)   (代码实现层)  (质量保障层)
```

**Step 3: Update Output directory structure (line 70-81)**

Add:
```
├── bug_reports/        # Bug Hunt reports per round
```

**Step 4: Add changelog entry (after line 128)**

```markdown
### v1.4 (2026-03-11)

- **Feature**: Bug Hunt phase — after development completes and summary.md is generated, an independent AI auditor scans all code against original requirements to find P0/P1/P2 bugs. Developer AI fixes each bug with regression tests. Loops until no new P2+ bugs found (max 15 rounds). Inner fix-verify retries reuse `AC_MAX_RETRIES` (default 10). Configurable via `{ENV_PREFIX}_BUG_HUNT_ROUNDS` env var.
```

**Step 5: Commit**

```bash
git add README.md
git commit -m "docs: update README with Bug Hunt phase"
```

---

### Task 6: Final verification

**Step 1: Read through the complete SKILL.md template flow**

Verify: config → ... → main() → cards loop → generate_summary() → run_bug_hunt() → done

**Step 2: Verify all function references exist**

Check that `run_bug_hunt` is defined before it's called in `main()`.

**Step 3: Verify template placeholders are consistent**

Ensure `{TEST_CMD}`, `{ENV_PREFIX}`, `{PROJECT_DISPLAY_NAME}`, `{AUTODEV_PATH}` are used consistently in the new code.

**Step 4: Push to remote**

```bash
git push
```
