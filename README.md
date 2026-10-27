# antithesis-skills-v2

Antithesis skills, packaged as one plugin for each agent platform.

- `plugins/antithesis/skills/antithesis/` is the entry point. Its `SKILL.md` routes `/antithesis <module>` to one module per subcommand. Every path inside the skill is written from the skill root, e.g. `references/triage/MODULE.md`.
- `plugins/antithesis/.devin-plugin/plugin.json` is the Devin manifest. Other platforms that read a manifest get their own next to it, pointing at the same `skills/`. Replit reads no manifest and imports `skills/` directly.

## Install

- Devin: [docs/devin.md](docs/devin.md)
- Replit: [docs/replit.md](docs/replit.md)


## Prerequisites

- You need an Antithesis container registry, credentials, and API key. 

## What this plugin does

This plugin enables a coding agent to use Antithesis. Setup, triage, and debugging happen exactly per Antithesis docs, but it takes a different approach to creating the harness around the SUT. 

The plugin will instruct your agent to: 

1. Package the SUT, deploy it to Antithesis, and initialize it in the Antithesis environment using `references/setup/` (this happens exactly per Antithesis docs).
2. Map all surfaces in the SUT by examining request handlers. 
3. Analyse the SUT and decide which property catalogs to use. 
4. If necessary, write a stateful client for the SUT that contains, e.g. retries, sessions, leases, etc. 
5. Create a driver to exercise the SUT. Instrument the SUT using the language's native coverage tools. The driver includes checks for property failures. 
6. Run the driver against the SUT, checking for property failures and code that never runs. Iterate locally until full coverage is achieved.
7. Iterate in Antithesis: launch a series of 5 minute runs to check instrumentation and coverage of code that runs only under faults (e.g. failovers).
8. Launch a 30 minute run, triage, and debug. (this happens exactly per Antithesis docs).

## Key differences from Antithesis docs

- Uses [Hegel](https://www.hegel.dev) as the generator that drives the system under test, not Antithesis' native action generator.
- Starts by trying to achieve full coverage of all surfaces in the system under test first, rather than reasoning about system properties.
- Generates a separate property catalog as its own object in the codebase. 

## Concepts / components not in official Antithesis docs

This plugin creates a *driver container* that it deploys to Antithesis alongside your system. The driver container is excluded from faults. It contains:

- A random generator driven by [Hegel](https://www.hegel.dev).
- Log listeners and checkers.
- Outcome listeners if the system has stateful clients.
- Relevant Property Catalogs, pre-written sets of invariants applicable to systems of a particular type. Each property is a file within the filesystem. 



