# Bug Hunt Phase Design

## Overview

Add a multi-round Bug Hunt phase after summary.md generation. The phase uses an independent AI auditor to scan code against requirements, find P0/P1/P2 bugs, fix them with regression tests, and loop until clean.

## Flow

```
summary.md generated → Bug Hunt Phase begins

Round N (N=1..BUG_HUNT_MAX_ROUNDS):
  [1] Bug Scan  — VERIFY_MODEL (auditor, read-only)
      Input: summary.md + requirements + source code + tests
      Output: bug_report_round_N.md (structured P0/P1/P2 list)

  [2] Check — parse report
      No P0/P1/P2 → EXIT success

  [3] Bug Fix — MODEL (developer)
      - Fix all reported bugs
      - Write regression test for each bug

  [4] Test — run full test suite (old + new)
      - Reuse TEST_MAX_RETRIES=10 auto-repair loop

  [5] Fix Verify — VERIFY_MODEL (auditor)
      - Verify each bug is actually fixed
      - Verify new tests cover the bugs
      - Fail → back to [3] (max AC_MAX_RETRIES=10 times)

  N++ → back to [1] full re-scan

Exit:
  - Normal: Round scan finds 0 P0/P1/P2
  - Safety cap: BUG_HUNT_MAX_ROUNDS reached → warning + remaining bug list
  - Final: update summary.md with Bug Hunt results
```

## Roles

| Role | Model | Responsibility |
|------|-------|----------------|
| Bug Hunter (auditor) | VERIFY_MODEL | Scan + verify fixes, read-only |
| Bug Fixer (developer) | MODEL | Fix code + write tests |

## Bug Report Format

```markdown
## Bug Report — Round N

### BUG-1 [P0] Short description
- File: path/to/file.py:line
- Problem: what's wrong
- Expected: what should happen
- Suggested fix: how to fix

### BUG-2 [P1] Short description
...
```

## Configuration

```bash
BUG_HUNT_MAX_ROUNDS="${{ENV_PREFIX}_BUG_HUNT_ROUNDS:-15}"
# Reuses existing:
# TEST_MAX_RETRIES (default 10) — test auto-repair
# AC_MAX_RETRIES (default 10) — fix verify inner retries
```

## File Layout

```
$AUTODEV/
  bug_reports/
    bug_report_round_1.md
    bug_report_round_2.md
    ...
  summary.md  (updated at end with Bug Hunt results)
```

## Integration Points

- Test verification: reuse existing test loop (timeout, auto-repair)
- Fix verify: same pattern as AC verification but targets bug report items
- Decisions audit: bug fix decisions appended to decisions.jsonl
- State tracking: bug hunt progress tracked in state file
