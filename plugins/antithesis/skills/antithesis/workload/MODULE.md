# Build a workload

Input: a system to test in Antithesis, and optionally the behaviors or properties to focus on.

A workload is the code that exercises the system during a run. It sends requests, checks the responses, and asserts properties. Antithesis injects faults around it and explores many timelines.

## How the workload is structured

Every workload this module builds has the same shape:

- **One action generator.** A Rust binary written with [hegel](https://hegel.dev) using `#[hegel::main]`. It picks actions at random, sends them to the system, and checks each response against a model of what the system should contain. Use hegel for all random choices, so Antithesis controls them.
- **An instrumented SUT.** Instrument the system under test with Antithesis coverage instrumentation, which guides the fuzzer.
- **Its own container.** The action generator runs alone in a `workload` container, which is excluded from faults.
- **Started by Test Composer.** The action generator is the test template's `singleton_driver_` command. Antithesis starts it once per timeline, after the system reports `setup_complete`. The action generator doesn't need to wait for the system to come up. It also checks that the system recovers once faults stop, in quiet periods it starts itself.

The action generator never exits on its own. Antithesis decides how long each timeline runs.

## Steps

1. **Map what the workload must reach.** Follow `references/surface.md`.
2. **Pick the model and properties** for every in-scope item. If a property catalog for this system exists, start from it.
3. **Instrument the SUT.** Follow `references/instrumentation.md`.
4. **Report setup complete** from the system's containers, once it can serve requests. See `references/harness.md`.
5. **Write the action generator.** Follow `references/action-generator.md`.
<!-- dev-only:start -->
   Also read `references/action-log.md`, and log every action as it describes.
<!-- dev-only:end -->
6. **Wire up the harness.** Add the `workload` container, the test template, and the fault exclusion. See `references/harness.md`.
7. **Build and validate.** Build the images, run the instrumentation checks, then `snouty validate`. Commit every file the images build from: check `git status --ignored`, since ignore rules can drop harness files.
8. **Iterate locally** on coverage and the SDK assertions. Follow `references/local-loop.md`.
9. **Iterate on short Antithesis runs.** Follow `references/run-loop.md`.
10. **Launch the final run**, 30 minutes long, and read it with the triage module.
11. **Report to the user** the bugs the workload found, any component left uninstrumented and why, and the coverage gaps you chose not to close.
