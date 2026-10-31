# antithesis-plugin

This plugin provides a coding agent to with an entry point to any task involving [Antithesis](https://www.antithesis.com/docs/). 

Generally, you'll want to include a prompt of the form "Use Antithesis to test this PR, then triage and debug any failures you find."

## Install

- Claude Code:
  - `claude plugin marketplace add antithesishq/antithesis-plugin`
  - `claude plugin install antithesis@antithesis`
- Codex:
  - `codex plugin marketplace add antithesishq/antithesis-plugin`
  - `codex plugin add antithesis@antithesis`
- Devin: [docs/devin.md](docs/devin.md)
- Replit: [docs/replit.md](docs/replit.md)

## Prerequisites

- You need an Antithesis container registry, credentials, and API key.

The plugin instructs your agent to check for, update, and install if necesssary:

- [Snouty CLI](https://github.com/antithesishq/snouty)
- [Docker](https://github.com/docker) or [Podman](https://github.com/podman-container-tools/podman), with Docker Compose.
- [jq](https://github.com/jqlang/jq)

## What this plugin does

Setup, triage, and debugging happen exactly per Antithesis docs, but it takes a different approach to creating the harness around the SUT.

The plugin will instruct your agent to:

1. Package the SUT, deploy it to Antithesis, and initialize it in the Antithesis environment using `references/setup/` (this happens exactly per Antithesis docs).
2. Map all surfaces in the SUT by examining request handlers.
3. Analyse the SUT and decide which property catalogs to use.
4. If necessary, write a stateful client for the SUT that contains, e.g. retries, sessions, leases, etc.
5. Create a driver to exercise the SUT. Instrument the SUT using the language's native coverage tools. The driver includes checks for property failures.
6. Run the driver against the SUT, checking for property failures and code that never runs. Iterate locally until full coverage is achieved.
7. Iterate in Antithesis: launch a series of 5 minute runs to check instrumentation and coverage of code that runs only under faults (e.g. failovers).
8. Launch a 30 minute run, triage, and debug. (this happens exactly per Antithesis docs).

### Key differences from Antithesis docs

- Uses [Hegel](https://www.hegel.dev) as the generator that drives the system under test, not Antithesis' native action generator.
- Starts by trying to achieve full coverage of all surfaces in the system under test first, rather than reasoning about system properties.
- Generates a separate property catalog as its own object in the codebase.

### Concepts / components not in official Antithesis docs

This plugin creates a *driver container* that it deploys to Antithesis alongside your system. The driver container is excluded from faults. It contains:

- A random generator driven by [Hegel](https://www.hegel.dev).
- Log listeners and checkers.
- Outcome listeners if the system has stateful clients.
- Relevant Property Catalogs, pre-written sets of invariants applicable to systems of a particular type. Each property is a file within the filesystem.

## Managing compute consumption

The following parameters are available to manage how much compute is used on a particular run, or set default levels of cumpute usage. All of these are also available in Snouty. 

We recommend cancelling a session once you're done to save compute hours. 

For a more complete discussion of compute hours, please see [the Antithesis docs on usage](https://antithesis.com/docs/configuration/usage/). 

### Setup

Setup time does not count towards compute consumption. Compute hours are counted from the time fuzzing starts.

`--setup-timeout 10m`

({{ t("--setup-timeout")? }}) sets a timeout for system setup in minutes, from 1m to 30m, defaulting to 5m. You must specify a whole number of minutes.

### Testing sessions

You can specify the maximum number of compute hours to be spent on a single test run, using 2 flags.

`--max-compute-hours 24`

{{ t("--max-compute-hours")? }} controls how many compute hours in total can be spent fuzzing, up to 1000 compute hours. Defaults to 48. 

`--performance-tier fast`

({{ t("--performance-tier")? }}) sets how many cores the run uses. Options are standard (48 cores), fast (96) or turbo (192), defaults to standard. 

After setup, Antithesis will fuzz for `max-compute-hours`/cores hours of wall-clock time. For example, `max-compute-hours 25` on `performance-tier standard` (48 cores) is 0.5h of fuzzing (and total test time will be 0.5h plus setup time). A faster tier spends the same budget on more cores, so it fuzzes for less wall-clock time.

The minimum value for `--max-compute-hours` is 1/4 of the tier's cores: 12 compute hours on standard, 24 on fast and 48 on turbo. Put another way, the shortest wall-clock run you can request is 0.25 hours (15 minutes).

### Debugging sessions

Starting with Release 67, you can specify the duration of debugging sessions in the Snouty CLI. If unspecified, length defaults to 1 hour.

`--duration 2h` 

The session stays available for the time {{ t("--duration")? }} gives, from 0.5 (30 min) to 6 hours.


