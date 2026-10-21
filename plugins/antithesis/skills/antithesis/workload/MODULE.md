# Build a workload

Input: a system to test in Antithesis, and optionally the behaviors or properties to focus on.

A workload is the code that exercises the system during a run. It sends requests, checks the responses, and asserts properties. Antithesis injects faults around it and explores many timelines.

Work without asking the user for input unless you are blocked.

## How the workload is structured

Every workload this module builds has the same shape:

- **One action generator.** A Rust binary written with [hegel](https://hegel.dev) using `#[hegel::main]`. It picks actions at random, sends them to the system, and checks each response against a model of what the system should contain. Use hegel for all random choices, so Antithesis controls them.
- **An instrumented SUT.** Every SUT binary is built with Antithesis coverage instrumentation, so Antithesis knows which code each timeline reaches.
- **Its own container.** The action generator runs alone in a `workload` container, which is excluded from faults. Faults still hit the system's own containers and the network between them.
- **Started by Test Composer.** The action generator is the test template's `singleton_driver_` command. Antithesis starts it once per timeline, after the system reports `setup_complete`. The action generator doesn't need to wait for the system to come up. A separate `eventually_` command checks that the system recovers once faults stop.

The action generator never exits on its own. Antithesis decides how long each timeline runs.

## Steps

1. **Map what the workload must reach.** Read `references/surface.md`. List every operation, option, background feature and behavior-changing setting the system has, decide for each one how the workload or deployment exercises it or why it's out of scope, and design the deployment around that. Write the table into the harness README.
2. **Pick the model and properties.** Decide what the model tracks and which properties to check against it, for every in-scope item. If a property catalog for this system already exists, start from that.
3. **Build the SUT with instrumentation.** Read `references/instrumentation.md` and instrument every SUT binary. This is required.
4. **Report setup complete.** Make the system emit `setup_complete` once it is ready to serve requests. This belongs in the system's containers, not in the action generator. See `references/harness.md`.
5. **Write the action generator.** Read `references/action-generator.md`. It has a complete example and the rules to follow.
<!-- dev-only:start -->
   Also read `references/action-log.md`, and log every action as it describes.
<!-- dev-only:end -->
6. **Wire up the harness.** Add the `workload` container, the test template with its `eventually_` command, and the fault exclusion. See `references/harness.md`.
7. **Check it locally.** Run the instrumentation checks from `references/instrumentation.md`, then `snouty validate` on the directory with `docker-compose.yaml`. Then run the action generator against `docker compose up` for a few minutes and check:
   - every action reaches its intended outcome at least sometimes (success, or the expected rejection for actions meant to fail), and none routinely takes much longer than the operation should
   - every sometimes assertion that doesn't depend on faults is reached
   - stopping a system container (`docker stop`) makes the system's process shut down, not get killed after the timeout

   Fix anything that fails. When a property fails, find out why before moving on, by reading the system's code path for that request:
   - If the system breaks something it promises, or that its real clients depend on, keep the property and record the bug in the harness README.
   - If the property demands more than the system promises by design, fix the property.

   Don't launch with properties you know are wrong: every false failure hides real ones.

   Finally, run `git status --ignored` and make sure every file the images build from is committed. The repository's ignore rules can silently drop harness files.
8. **Close coverage gaps.** Launch a run, then use its coverage to find code the workload never reached, and fix the workload or deployment. Repeat until the remaining gaps are out of scope or rare. See `references/coverage-loop.md`.
