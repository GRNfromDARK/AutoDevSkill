# autodev-generator

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that generates complete **Autodev automated development pipelines** from a structured todolist.md file.

## What it does

Give it a todolist → get a fully automated, gated, TDD-enforced development pipeline that runs itself.

```
todolist.md  →  Autodev/{project}/
                 ├── autodev.sh          # Main pipeline script
                 ├── system_prompt.md    # AI session prompt
                 ├── gate_check.sh      # Automated gate checks
                 ├── cards/*.md         # Task cards
                 ├── state              # Progress tracker
                 └── decisions.jsonl    # Decision audit trail
```

Then run:

```bash
./Autodev/{project}/autodev.sh
```

The pipeline drives Claude Code through card-by-card development — fully automated, no human in the loop.

## Key Features

- **TDD-enforced**: Every card follows RED → GREEN → SPEC → LINT → GATE
- **Auto-repair**: Failed tests trigger AI self-fix (up to 3 retries)
- **Gated phases**: Automated gate checks between development phases
- **Independent verification**: Separate AI verifies acceptance criteria per card
- **AI mutual review**: High-risk decisions get independent AI review (no human blocking)
- **Decision audit trail**: `decisions.jsonl` tracks all decisions across cards
- **Resumable**: State file tracks progress; restart from any card with `--from`
- **Pipeline summary**: Auto-generates structured report on completion

## Install

```bash
npx skills add anthropic-rex/autodev-generator
```

## Usage

In Claude Code:

```
帮我生成 probability_engine 的 autodev 文件
```

or

```
create autodev for my-feature based on todolist.md
```

The skill reads your todolist, gathers project context, designs the pipeline (phases, cards, gates), and generates all files.

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

Each **Card** runs one Claude Code session:
1. Reads spec + todolist + existing code
2. Implements via TDD (test first → implement → verify)
3. Auto-repairs if tests fail
4. Independent AI verifies acceptance criteria
5. Records decisions to `decisions.jsonl`

Each **Gate** runs automated checks:
1. Unit tests + full regression
2. Decision audit (SPEC-DECISION / AI-REVIEW annotations)
3. Cross-file change coverage
4. AI compliance audit

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
- A `todolist.md` with grouped tasks (the skill reads this to generate the pipeline)
- Optionally, a spec/design document referenced by the todolist

## License

MIT
