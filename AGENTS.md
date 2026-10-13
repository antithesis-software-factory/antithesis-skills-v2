# AGENTS.md

This repo packages the Antithesis skills as one plugin, `plugins/antithesis`, for each agent platform.

## Paths

- Skills: `plugins/antithesis/skills/`
- Entry-point skill: `plugins/antithesis/skills/antithesis/SKILL.md`
- Modules: `plugins/antithesis/skills/antithesis/<module>/MODULE.md`
- Devin manifest: `plugins/antithesis/.devin-plugin/plugin.json`
- Platform setup docs: `plugins/antithesis/docs/`

## Rules

- `plugins/antithesis/skills/` is the only copy of the skills. Don't copy them into another repo or into a platform-specific folder. A new platform gets a manifest in `plugins/antithesis/`, not new skills.
- `triage`, `debug`, `workload` and `properties` are modules of the single `antithesis` skill. Don't split them into separate skills. Add a new subcommand as a module and a row in the `SKILL.md` routing table.
- Keep `SKILL.md` to routing. Detail goes in the module's `MODULE.md`, and long reference material in `<module>/references/`.
- Core skills don't branch on platform. Platform-specific runtime notes go in `plugins/antithesis/skills/antithesis/platforms/<platform>.md`, linked from the Platform notes section of `SKILL.md`. Problems that happen before the skill loads (installing or invoking it) can't be fixed in the skill; document those in `plugins/antithesis/docs/<platform>.md`, in the prompt that starts the session.
- Bump `version` in each platform manifest when the skills change in a way users should pick up.
