# Closing coverage gaps

Your surface table is a guess about what the workload reaches. The run's coverage says what it actually reached. Use it to find what the table missed.

## Read a run's coverage

Once a run has finished, its coverage is in the run's results:

```bash
snouty --json runs show <run_id> > run.json
jq '.results.coverage.files | keys' run.json
```

`.results.coverage.files` maps each instrumented source file to the functions in it that ran at least once, each with the source spans reached (`[start_line, start_col, end_line, end_col]`). A file or function that never ran is simply absent. If the run has no coverage for the system's language, skip this loop and say so in the README. To see what's missing, compare against the system's source tree:

1. **Modules, packages or directories with no reached file at all.**
2. **Files that were reached, but where many of their functions are missing.**
3. **Functions that ran, but where whole branches are outside every reached span.** Look at these in the biggest and most central functions: request handlers, the storage layer, the replication or coordination code.

## Classify each gap

For every gap of any size, decide which of these it is:

- **Out of scope:** code for a deployment this harness doesn't use, stubs that only reject requests, test helpers. Add it to the surface table's out-of-scope list if it isn't there.
- **A request the workload never sends:** an operation, an option, or a combination of them. Add it to the action generator.
- **A feature the deployment never turns on:** a background job that's off by default, a configuration the nodes don't use, a topology the harness lacks. Change the deployment.
- **A condition that's rare:** an error path or race that needs a specific fault or timing. Only act on this if a different action mix or weighting would make the condition common. Don't chase individual error paths.

## Loop

1. Launch a run and wait for it to finish.
2. Read its coverage and classify the gaps as above.
3. Fix the workload and deployment for every gap that's a missing request or a missing feature, and update the surface table.
4. Check locally again, then launch the next run.

Stop when the remaining gaps are all out of scope or rare conditions, or after three runs. In the harness README, keep a short list of the gaps you found and what you changed for each.
