# robisie agent skills

Installable [Agent Skills](https://agentskills.io) mirrored from [robisie.app](https://robisie.app).
Give an AI agent the procedures to run a project on a live board: take the next card, return evidence, and keep state on the board instead of in chat.

Live pages on robisie.app are the source of truth. This repo exists so `npx skills add` can install them.

## Install

```bash
npx skills add chuppa12/robisie-agent-skills
```

One skill:

```bash
npx skills add chuppa12/robisie-agent-skills --skill robisie -a claude-code
npx skills add chuppa12/robisie-agent-skills --skill robisie-project-steward -a claude-code
npx skills add chuppa12/robisie-agent-skills --skill robisie-acceptance-bench -a claude-code
```

List without installing:

```bash
npx skills add chuppa12/robisie-agent-skills --list
```

## Procedures

- **`robisie`** — worker loop: read the board, claim the next unblocked card, do the work, attach evidence, submit `in_review`. Live: <https://robisie.app/SKILL.md>
- **`robisie-project-steward`** — lead the whole project on the live board (atomic cards, dependencies, evidence, one decision bundle). Live: <https://robisie.app/skills/project-steward/SKILL.md>
- **`robisie-acceptance-bench`** — accept or reject finished agent work against evidence. Live: <https://robisie.app/skills/acceptance-bench/SKILL.md>

Each `SKILL.md` is a byte-faithful mirror of its live page (plus one HTML comment pointing at the source). Do not edit the copies here by hand.

## License

[MIT](LICENSE)
