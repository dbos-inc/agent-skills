# DBOS Agent Skills

Agent Skills help developers use AI agents to add DBOS durable workflows to their applications.
The skills in this repository can automatically be used by coding agents like Claude Code or Cursor to teach them how to use DBOS.

The skills in this repo follow the [Agent Skills](https://agentskills.io/) format.

## Installation

```bash
npx skills add dbos-inc/agent-skills
```

## Skill Structure

Each skill follows the [Agent Skills Open Standard](https://agentskills.io/):

- `SKILL.md` - Required skill manifest with frontmatter (name, description, metadata)
- `AGENTS.md` - Compiled references document (generated)
- `references/` - Individual reference files
