# antithesis-skills-v2

Platform-neutral Antithesis skills.

- `skills/antithesis/` is the entry point. `SKILL.md` routes to one module per subcommand.
- `skills/snouty/` sets up and uses snouty, the Antithesis CLI.

Each agent platform gets its own plugin repo that copies these skills in at a pinned commit and adds whatever that platform needs.
