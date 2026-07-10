# Supporting Development Skills

The legacy `auto-dev` pipeline generator has been retired and removed from this repository.

The repository continues to contain the independent development practices that were previously bundled
alongside it:

| Skill | Purpose |
|-------|---------|
| `test-driven-development` | Test-first behavior development |
| `systematic-debugging` | Evidence-based root-cause investigation |
| `verification-before-completion` | Completion claims backed by current verification |
| `dispatching-parallel-agents` | Parallel work for genuinely independent tasks |
| `brainstorming` | Design exploration |
| `subagent-driven-development` | Scoped subagent implementation and review |
| `requesting-code-review` | Independent code review workflow |

Install the repository's remaining skills with:

```bash
npx skills add GRNfromDARK/AutoDevSkill --all
```

Or copy only the desired directory from `skills/` into your agent's skill directory.

## Removal Note

Removed on 2026-07-10:

- `skills/auto-dev/`
- auto-dev-specific pipeline templates
- auto-dev-specific Bug Hunt design documents

The removal does not affect the seven standalone skills listed above.

## License

MIT
