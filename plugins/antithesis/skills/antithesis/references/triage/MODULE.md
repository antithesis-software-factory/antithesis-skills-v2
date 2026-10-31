# Triage an Antithesis run

Input: an optional mode, then an Antithesis run ID. If you were given something else, like a triage report URL or "the latest run", read `references/triage/run-discovery.md` to find the run ID.

Work through the run without asking the user for input unless you are blocked.

The `references/triage/` directory has detailed guides. Read each one only when a step points to it.

## Setup

Set up snouty with `references/antithesis/snouty-setup.md`. You also need `jq`.

Check that snouty can reach Antithesis:

```bash
snouty doctor --json
```

Continue only if the top-level `ok` is `true` and the `api_key` check's `status` is `ok`. Otherwise report the failing check's `message` and stop.

## Find the run

`snouty --json runs show <run_id>` gives the run's status, metadata and links.

- If the run can't be found, wait 10 seconds and retry. A run can take a while to appear in the API after it starts.
- If the status is `starting`, wait for it with `snouty runs wait --until ready <run_id>`.
- If the status is `incomplete`, follow "Diagnose an incomplete run" below instead.

## Pick a mode

| Mode         | Read                              | Use when                                                                              |
| ------------ | --------------------------------- | ------------------------------------------------------------------------------------- |
| `monitor`    | `references/triage/monitor.md`    | The run is in progress. Report and triage failures as they arrive.                    |
| `summary`    | `references/triage/summary.md`    | The run has finished. Summarize its failures and what's new.                          |
| `root-cause` | `references/triage/root-cause.md` | One failing property or anomaly, in a running or finished run, needs its root cause. |

If no mode is given, use `monitor` for an `in_progress` run and `summary` for a finished one.

## Read the properties

`snouty --json runs properties <run_id>` lists every property, one JSON object per line. Failing ones have `"status": "Failing"` and example moments. Read `references/triage/properties.md` for the format, the assertion types, and what pass/fail ratios suggest.

If the run is `in_progress`, don't wait for it to finish. Follow `snouty runs stream --help` to get the current results and new ones as they arrive.

If a `completed` or `cancelled` run has no properties, it has no results to triage. Report that and stop.

Also check the `Software was instrumented` and `Symbols were uploaded` properties, and include the instrumentation status in your report. Read `references/triage/instrumentation.md` to interpret them.

## Investigate a failing property

Read `references/triage/logs.md` before downloading any logs. Then:

1. Pick the first failing example and find its moment. If the counterexample has no `moment` (telemetry and meta properties, see `references/triage/properties.md`), the counterexample value is the evidence; skip to step 4.
2. Download the log for that moment and analyze it locally.
3. If you aren't sure what caused the failure, find more moments for the same property with `snouty runs search -n 999` and compare their logs. Logs from passing examples are useful for finding what differs between success and failure.
4. Cross-reference the log against the source code of the system under test and the workload.
5. Work out the timeline of events that led up to the failure and its root cause.

The property status and assertion text alone are not enough. Download and read the logs before concluding on any failure that has moments. If a property has a "details" section, use it: its author chose those fields to show the system's state at the failure.

## Diagnose an incomplete run

1. Look at `failure_moment` in `snouty --json runs show <run_id>`. If present, download a log for its `input_hash` and `vtime` as described in `references/triage/logs.md`.
2. Get the build logs with `snouty --json runs build-logs <run_id>`, especially if there is no failure moment, and look for errors.

For incomplete runs the per-property endpoints usually return 404, but `links.triage_report` and `failure_moment` may still be set. Report what `show` actually contains.

## Report

Follow `references/triage/issues.md` to report what you found.

## Before you report

Check each report against these:

- Every property status you report came from the properties data or the stream, not inference.
- Conclusions about failures with moments rest on evidence from the run, with specific log lines or command output, timestamps and property names.
- The report separates what the run shows from what you interpret.
- A claim that a binary isn't instrumented follows `references/triage/instrumentation.md` and doesn't rest only on the deduplicated examples of `Software was instrumented`.
