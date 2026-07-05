# robisie agent skills

Installable [Agent Skills](https://agentskills.io) that hand your AI agent the operating
procedures behind [robisie.app](https://robisie.app) — so your agent can **run the project
itself**: self-onboard onto an agent-readable board and take delegated work end-to-end,
without an operator translating tickets between steps.

## Install

Add every skill in this repo:

```bash
npx skills add chuppa12/robisie-agent-skills
```

Or install one skill for a specific agent (e.g. Claude Code):

```bash
npx skills add chuppa12/robisie-agent-skills --skill robisie-planer -a claude-code
```

List what's here without installing:

```bash
npx skills add chuppa12/robisie-agent-skills --list
```

## Procedures

- **`robisie-planer`** — self-onboard onto the Robisie Planer, an agent-readable kanban board
  exposed over MCP. Read the board, take the next unblocked task by priority, do it, and report
  progress — you self-direct from the board instead of an operator translating tickets for you.
  Live source of truth: <https://robisie.app/SKILL.md>
- **`robisie-acceptance-bench`** — the acceptance protocol for handing finished agent work back
  with evidence. It separates "the agent claims it's done" from "a human accepted it", and moves
  the hand-off outside the frame of code and pull requests.
  Live source of truth: <https://robisie.app/skills/acceptance-bench/SKILL.md>

Each `SKILL.md` here is a byte-faithful mirror of its live page on robisie.app (the only line
added is a comment pointing back at the source). The URLs above are the canonical, always-current
versions.

## License

[MIT](LICENSE).
