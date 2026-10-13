---
name: snouty
description: Set up and use snouty, the Antithesis CLI, to inspect Antithesis runs, their properties, and logs. Use before running any snouty command.
---

# snouty

## Install

Run once per session, before the first snouty command:

```bash
if ! command -v snouty >/dev/null 2>&1; then
  curl --proto '=https' --tlsv1.2 -LsSf https://github.com/antithesishq/snouty/releases/latest/download/snouty-installer.sh | SNOUTY_NO_MODIFY_PATH=1 sh
  export PATH="$HOME/.local/bin:$PATH"
fi
snouty version
```

## Credentials

snouty requires `ANTITHESIS_TENANT` and `ANTITHESIS_API_KEY` to be set. If either is missing, ask the user to add it to the environment (for example, their agent platform's secrets). Don't use `snouty login`. `snouty doctor` checks the setup.

## Commands

Pass `--json` when parsing output.

- `snouty runs list`: recent runs.
- `snouty runs show <run_id>`: status and links for one run.
- `snouty runs wait <run_id>`: block until the run finishes.
- `snouty runs properties <run_id>`: each property and whether it passed. Failing properties include example moments.
- `snouty runs logs <run_id> <hash> <vtime>`: logs for one timeline up to a moment. Always pass `vtime`; without it the log streams to the end of the timeline.
- `snouty runs events <run_id> -m <needle>`: events in the run that contain a string.
- `snouty runs search <run_id> <query>`: an event-set DSL query over the run's events. See `snouty runs search --help`.
