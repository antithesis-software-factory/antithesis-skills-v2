# antithesis-skills-v2

Antithesis skills, packaged as one plugin for each agent platform.

- `plugins/antithesis/skills/antithesis/` is the entry point. Its `SKILL.md` routes `/antithesis <module>` to one module per subcommand.
- `plugins/antithesis/skills/snouty/` sets up and uses snouty, the Antithesis CLI.
- `plugins/antithesis/.devin-plugin/plugin.json` is the Devin manifest. Other platforms get their own manifest next to it, pointing at the same `skills/`.

## Install

Devin: Customize → Plugins → Add plugin → From repository. Enter this repo and the subdirectory `plugins/antithesis`. [docs/devin.md](plugins/antithesis/docs/devin.md) covers the rest of customer setup.
