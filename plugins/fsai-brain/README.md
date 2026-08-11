# fsai-brain

Skills for writing into the [fsai-brain vault](https://github.com/franchiseai/fsai-brain) without breaking its structure.

Design rule: **skills exist only for ingestion**. Reading the brain, and free-form writing inside it, need no skill; the vault's `CLAUDE.md` carries the conventions (frontmatter spine, feature states, the fog rule). Skills cover the two paths where structure must survive many writers:

| Skill | Purpose |
|---|---|
| `/fsai-brain:capture` | Dev end-of-day sweep: turn today's commits, PRs, and session decisions into ledger updates, decisions, and queue items. |
| `/fsai-brain:ingest <text>` | Route any free text (a thought, a Slack thread, "we decided X") into the right note types with correct frontmatter and links. |

The daily Slack digest is not a skill; it is `scripts/daily-digest.sh` in the vault, run from cron on the VPS.

Vault location: `~/code/fsai-brain` by default; override with the `FSAI_BRAIN_PATH` environment variable.
