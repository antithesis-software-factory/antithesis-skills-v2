# Build a workload

Input: a system to test in Antithesis, and optionally the behaviors or properties to focus on.

A workload is the code that exercises the system during a run. It sends requests, checks the responses, and asserts properties. Antithesis injects faults around it and explores many timelines.

Work without asking the user for input unless you are blocked.

## How the workload is structured

Every workload this module builds has the same shape:

- **One action generator.** A Rust binary written with [hegel](https://hegel.dev) using `#[hegel::main]`. It picks actions at random, sends them to the system, and checks each response against a model of what the system should contain. Use hegel for all random choices, so Antithesis controls them.
- **An instrumented SUT.** Every SUT binary is built with Antithesis coverage instrumentation, so Antithesis knows which code each timeline reaches.
- **Its own container.** The action generator runs alone in a `workload` container, which is excluded from faults. Faults still hit the system's own containers and the network between them.
- **Started by Test Composer.** The action generator is the test template's only command, a `singleton_driver_`. Antithesis starts it once per timeline, after the system reports `setup_complete`. The action generator doesn't need to wait for the system to come up.

The action generator never exits on its own. Antithesis decides how long each timeline runs.

## Steps

1. **Pick what to test.** List the actions the workload will take, usually the system's public API calls. Decide what the model tracks and which properties to check against it. If a property catalog for this system already exists, start from that.
2. **Build the SUT with instrumentation.** Read `references/workload/instrumentation.md` and instrument every SUT binary. This is required.
3. **Report setup complete.** Make the system emit `setup_complete` once it is ready to serve requests. This belongs in the system's containers, not in the action generator. See `references/workload/harness.md`.
4. **Write the action generator.** Read `references/workload/action-generator.md`. It has a complete example and the rules to follow.
5. **Wire up the harness.** Add the `workload` container, the test template, and the fault exclusion. See `references/workload/harness.md`.
6. **Check it locally.** Run the instrumentation checks from `references/workload/instrumentation.md`, then `snouty validate` on the directory with `docker-compose.yaml`. Fix anything either reports.
