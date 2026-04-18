# Stategraph skills

Agent skills for working with [Stategraph](https://stategraph.com).

This repo packages five skills compatible with [skills.sh](https://skills.sh), Claude Code, and other agents that load skills from `SKILL.md` files.

## Skills

| Skill | Purpose |
| --- | --- |
| `stategraph` | Router. Detects Stategraph tasks and dispatches to one of the four workflow skills below. |
| `stategraph-query` | Read-only MQL queries, state summaries, inventory, blast radius, and gap analysis. |
| `stategraph-change` | `stategraph tf plan`, `stategraph tf apply`, state deletion, transaction lifecycle. |
| `stategraph-import` | Import `.tfstate` or HCL into Stategraph; wire a Terraform repo to Stategraph. |
| `stategraph-refactor` | Interactive address-rewrite workflow — restructure a repo while preserving state addresses. |

The router skill hands off to the others via the agent's Skill tool, so users typically just need the top-level `stategraph` skill installed and the subskills are invoked automatically. Installing all five together also makes each workflow directly addressable (e.g. `/stategraph-query`).

## Install

With the [skills CLI](https://skills.sh):

```bash
npx skills add stategraph/skills
```

Or install a specific skill:

```bash
npx skills add stategraph/skills -s stategraph-query
```

### Manual install (Claude Code)

Clone into `~/.claude/skills/` so each skill lands at `~/.claude/skills/<name>/SKILL.md`:

```bash
git clone https://github.com/stategraph/skills /tmp/stategraph-skills
cp -r /tmp/stategraph-skills/skills/* ~/.claude/skills/
rm -rf /tmp/stategraph-skills
```

## Usage

Once installed, ask your agent anything Stategraph-related and the router will pick the right workflow:

- "What S3 buckets do we have?" → `stategraph-query`
- "Plan these changes." → `stategraph-change`
- "Import this terraform.tfstate." → `stategraph-import`
- "Refactor this repo into child modules without losing state." → `stategraph-refactor`

You can also invoke a workflow directly:

```
/stategraph-query
/stategraph-change
/stategraph-import
/stategraph-refactor
```

## Requirements

- The [`stategraph` CLI](https://stategraph.com) on your `PATH`.
- A configured Stategraph tenant (`stategraph info` should succeed).

## License

MIT
