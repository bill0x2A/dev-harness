# fsai Claude Plugin Marketplace

Claude Code plugins for FSAI development.

## Install

```bash
claude plugin marketplace add bill0x2A/dev-harness
claude plugin install fsai-dev@dev-harness
```

## Plugins

### fsai-dev

A composable development pipeline. Development processes (research, spec grilling, planning, wave implementation, architecture and design-system checks, backend testing, PR creation) are packaged as **phase skills** with a standard contract (inputs, artifacts, exit criteria, gate). A per-task **pipeline manifest** records what ran, what was skipped and why, and every decision. The `/fsai-dev:feature` conductor proposes a pipeline for a task, runs it, and stops only at approval gates.

Start here: [plugins/fsai-dev/README.md](plugins/fsai-dev/README.md)

## Contributing

Each phase skill must conform to the phase contract (`plugins/fsai-dev/docs/phase-contract.md`). Add new phases as skills, register them in the catalog table, and keep repo-specific knowledge clearly labeled with the repo it applies to.
