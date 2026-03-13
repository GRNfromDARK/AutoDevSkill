# gate_check.sh Template

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
spec_count=$(grep -r "SPEC-DECISION" {SOURCE_DIRS} 2>/dev/null | wc -l | tr -d ' ' || echo 0)
review_count=$(grep -r "AI-REVIEW" {SOURCE_DIRS} 2>/dev/null | wc -l | tr -d ' ' || echo 0)
echo "  SPEC-DECISION 标注: $spec_count 处"
echo "  AI-REVIEW 标注: $review_count 处"

# 4. AI-REVIEW 覆盖率检查
# 检查跨文件变更是否有对应的 AI-REVIEW 记录
if [ -f "$AUTODEV_DIR/decisions.jsonl" ] && [ -s "$AUTODEV_DIR/decisions.jsonl" ]; then
    block_count=$(grep -c '"severity": *"BLOCK"' "$AUTODEV_DIR/decisions.jsonl" 2>/dev/null || echo 0)
    warn_count=$(grep -c '"severity": *"WARN"' "$AUTODEV_DIR/decisions.jsonl" 2>/dev/null || echo 0)
    total_decisions=$(wc -l < "$AUTODEV_DIR/decisions.jsonl" | tr -d ' ' || echo 0)
    review_decisions=$(grep -c '"level": *"AI-REVIEW"' "$AUTODEV_DIR/decisions.jsonl" 2>/dev/null || echo 0)
    echo "  decisions.jsonl: $total_decisions 条记录 (AI-REVIEW: $review_decisions, BLOCK: $block_count, WARN: $warn_count)"
    # 检查是否有未解决的 BLOCK
    unresolved_blocks=$(grep '"severity": *"BLOCK"' "$AUTODEV_DIR/decisions.jsonl" 2>/dev/null | grep '"consensus": *false' | wc -l | tr -d ' ' || echo 0)
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
    changed_files=$(git diff --name-only "$phase_base_ref" -- 2>/dev/null | wc -l | tr -d ' ' || echo 0)
    echo "  Phase 基线: $phase_base_ref"
    phase_review_count=0
    while IFS= read -r changed_file; do
        [ -f "$changed_file" ] || continue
        file_review_count=$(grep -c "AI-REVIEW" "$changed_file" 2>/dev/null || true)
        phase_review_count=$((phase_review_count + file_review_count))
    done < <(git diff --name-only "$phase_base_ref" -- 2>/dev/null)
else
    changed_files=$(git diff --name-only HEAD 2>/dev/null | wc -l | tr -d ' ' || echo 0)
    phase_review_count=$review_count
    check_warn "未找到有效 .phase_baseline，暂回退到工作区 diff；建议每个 Gate 成功后更新该基线"
fi
if [ "$changed_files" -gt 3 ] && [ "$phase_review_count" -eq 0 ]; then
    check_warn "本 Phase 变更了 $changed_files 个文件但代码中无 AI-REVIEW 标注"
else
    check_pass "AI-REVIEW 覆盖率正常 ($changed_files 文件变更, 本 Phase $phase_review_count 处 AI-REVIEW 标注)"
fi

# ──── 路径边界检查 ────
# 检查是否有越界修改（只允许修改 {ALLOWED_PATHS} 下的文件）
# 生成时请将 {ALLOWED_PATHS} 替换为允许修改的目录列表（空格分隔），
# 将 {FORBIDDEN_PATHS} 替换为禁止修改的文件/目录列表（空格分隔），
# 如果项目不需要路径限制，删除此段。
ALLOWED_PATHS="{ALLOWED_PATHS}"
FORBIDDEN_PATHS="{FORBIDDEN_PATHS}"
if [ -n "$ALLOWED_PATHS" ] && [ "$ALLOWED_PATHS" != "{""ALLOWED_PATHS""}" ]; then
    echo ""
    echo "── 路径边界检查 ──"
    boundary_ok=true
    while IFS= read -r changed_file; do
        in_scope=false
        for allowed in $ALLOWED_PATHS; do
            case "$changed_file" in "$allowed"*) in_scope=true; break ;; esac
        done
        if [ "$in_scope" != true ]; then
            check_fail "越界修改: $changed_file (不在允许范围 [$ALLOWED_PATHS] 内)"
            boundary_ok=false
        fi
    done < <(git diff --name-only "${phase_base_ref:-HEAD~1}" -- 2>/dev/null)
    [ "$boundary_ok" = true ] && check_pass "所有变更文件在允许路径范围内"
fi
if [ -n "$FORBIDDEN_PATHS" ] && [ "$FORBIDDEN_PATHS" != "{""FORBIDDEN_PATHS""}" ]; then
    for forbidden in $FORBIDDEN_PATHS; do
        if git diff --name-only "${phase_base_ref:-HEAD~1}" -- 2>/dev/null | grep -q "^${forbidden}"; then
            check_fail "禁止修改的路径被改动: $forbidden"
        fi
    done
fi

# ──── 项目特定检查 (定制) ────
# {PROJECT_SPECIFIC_CHECKS}

# ──── 汇总 ────
echo ""
echo "═══════════════════════════════════════════════"
echo "结果: ✅ $PASS | ❌ $FAIL | ⚠️  $WARN"
[ $FAIL -gt 0 ] && exit 1 || exit 0
```
