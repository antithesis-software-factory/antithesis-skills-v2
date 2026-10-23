# Iterating on short runs

## 1. Launch a short run

Launch a 5-minute run with the command in `references/workload/harness.md`, and read it with the triage module.

## 2. Check the results

- **Instrumentation.** `Software was instrumented` and `Every instrumented module has symbols` pass. If not, fix the build before reading anything else.
- **Failing properties.** Early failures are usually the model being stricter than the system under faults, such as treating a timed-out write as lost. Apply the same rule as locally: keep a property that catches the system breaking a promise, and fix one that demands more than the system promises.
- **Behavior that needs faults.** Sometimes assertions that need faults are reached, and quiet periods start, recover and pass.
- **Client failures**, with client containers. Clients restart and get partitioned during runs, and the action generator handles it. A reachable assertion where it notices a new boot makes this easy to check.
- **Coverage** of code only faults reach, and of services local coverage couldn't measure:
  ```bash
  snouty --json runs show <run_id> > run.json
  jq '.results.coverage.files | keys' run.json
  ```
  For Go, Rust and C/C++, `.results.coverage.files` maps each instrumented file, by its build path, to the functions that ran, with the spans reached (`[start_line, start_col, end_line, end_col]`). Code that never ran is absent.

## 3. Fix and repeat

Run the local loop again after every change, then launch the next run. A few runs are usually enough.

A 5-minute run reaches little of the system's state, so don't worry too much about failing sometimes assertions for rare conditions. Common sometimes assertions should still fire.
