# Build a workload

Input: a system to test in Antithesis, and optionally the behaviors to focus on.

## How the workload is structured

Every workload this module builds has these parts:

- **One driver.** A Rust binary written with [hegel](https://hegel.dev) using `#[hegel::main]`. It picks actions at random, has them sent to the system, and records each outcome in a model of what the system should contain. Use hegel for all random choices, so Antithesis controls them.
- **An instrumented SUT.** Instrument the system under test with Antithesis coverage instrumentation.
- **The system's logs, forwarded to the driver.** Each system container's logs are forwarded to the driver, which parses them into each node's state.
- **Its own container.** The driver runs alone in a container excluded from faults, so it can always act and see everything.
- **Started by Test Composer.** The driver is the test template's `singleton_driver_` command. Antithesis starts it once per timeline, after the system reports `setup_complete`. The driver doesn't need to wait for the system to come up. It also starts quiet periods, which pause faults so the system can recover.

The system's real client library runs in client containers that are faulted like the servers, and the driver tells them what to do. Skip them only if the system has no client library, or its client keeps nothing between requests. See `references/workload/clients.md`.

The driver never exits on its own. Antithesis decides how long each timeline runs.

## Steps

1. **Map what the workload must reach.** Follow `references/workload/surface.md`.
2. **Pick the model** for every in-scope item: what it must record for a property to judge each response.
3. **Instrument the SUT**, including the client library. Follow `references/workload/instrumentation.md`.
4. **Report setup complete** from the system's containers, once it can serve requests. See `references/workload/harness.md`.
5. **Write the client program.** Follow `references/workload/clients.md`.
6. **Write the driver.** Follow `references/workload/driver.md`.
7. **Wire up the harness.** Add the `driver` container, the client containers, the test template, and the fault exclusion. See `references/workload/harness.md`.
8. **Build and validate.** Build the images, run the instrumentation checks, then `snouty validate`. Commit every file the images build from: check `git status --ignored`, since ignore rules can drop harness files.
9. **Iterate locally** on coverage and the exploration assertions. Follow `references/workload/local-loop.md`.
10. **Forward the system's logs.** Follow `references/workload/system-logs.md`, then iterate locally again.
11. **Iterate on short Antithesis runs.** Follow `references/workload/run-loop.md`.
12. **Launch the final run** with `--max-compute-hours 24` on the standard tier (30 minutes of fuzzing). Follow `references/launch/MODULE.md`, and read the run with the triage module.
13. **Report to the user**.
