# antithesis-skills-v2

Antithesis skills, packaged as one plugin for each agent platform.

- `plugins/antithesis/skills/antithesis/` is the entry point. Its `SKILL.md` routes `/antithesis <module>` to one module per subcommand. Every path inside the skill is written from the skill root, e.g. `references/triage/MODULE.md`.
- `plugins/antithesis/.devin-plugin/plugin.json` is the Devin manifest. Other platforms that read a manifest get their own next to it, pointing at the same `skills/`. Replit reads no manifest and imports `skills/` directly.

## Install

- Devin: [docs/devin.md](docs/devin.md)
- Replit: [docs/replit.md](docs/replit.md)
