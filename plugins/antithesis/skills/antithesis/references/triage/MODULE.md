# Triage an Antithesis run

Input: an Antithesis run ID. If you were given something else, like a triage report URL or "the latest run", read `references/triage/run-discovery.md` to find the run ID.

Work through the run without asking the user for input unless you are blocked.

The `references/triage/` directory has detailed guides. Read each one only when a step below points to it.

## Setup

Set up snouty with `references/antithesis/snouty-setup.md`. You also need `jq`.

Check that snouty can reach Antithesis:

```bash
snouty doctor --json
```

Continue only if the top-level `ok` is `true` and the `api_key` check's `status` is `ok`. Otherwise report the failing check's `message` and stop.

## Find failing properties

`snouty --json runs show <run_id>` gives the run's status, metadata and links.

- If `links.triage_report` is null or absent, no report was generated (typical for `cancelled` runs and some `unknown` or `starting` states). Report that the run can't be triaged; the properties and logs endpoints return 404 for these runs.
- If the status is `incomplete`, follow "Diagnose an incomplete run" below instead.

`snouty --json runs properties <run_id>` lists every property, one JSON object per line. Failing ones have `"status": "Failing"` and example moments. Read `references/triage/properties.md` for the format, the assertion types, and what pass/fail ratios suggest.

If the run is `in_progress`, don't wait for it to finish. Follow `snouty runs stream --help` to get the current results and new ones as they arrive.

If the run can't be found, wait 10 seconds and retry. A run can take a while to appear in the API after it starts.

Also check the `Software was instrumented` and `Symbols were uploaded` properties, and include the instrumentation status in your report. Read `references/triage/instrumentation.md` to interpret them.

## Investigate failed properties

Triage each failing property in its own subagent.

Read `references/triage/logs.md` before downloading any logs. Then, for each failing property:

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

## Report issues

Report one issue for each root cause among the failed properties. Do not report multiple issues for the same root cause, even if multiple properties failed.

By default, use the following format for reporting issues. If the user has specified a particular place or format of how to report issues, use that instead. If the user has not specified where to report issues, print the report in the session instead. The following format is written with GitHub in mind; adapt any GitHub-specific syntax for other destinations as appropriate.

Write for a maintainer who knows this codebase but not Antithesis or this run. Lead with what's wrong and where. Explain one representative occurrence in the code's terms. Put supporting evidence in a collapsed `<details>` block. Don't describe how you investigated.

```
# <title>

<antithesis property name that failed>

<description of bug>

## Reproduction

<reproducer>

## Details

* Antithesis run: `<run id>`
* Input hash: `<input hash>`
* vtime: `<vtime>`
* Version: <git hash>

<duplicate_details>

<!-- antithesis-triage-fingerprint
<fingerprint>
-->
```

### Reproducer

Write a reproducer for the bug and put it in <reproducer>. A reproducer is code that the maintainer can run to reproduce the bug that does not involve Antithesis. Prose explanations or instructions are not reproducers. Run the reproducer and ensure it reproduces the bug before including it in the report. Don't commit it to the repository.

If writing a reproducer would be too hard, don't write one, and instead silently leave out the Reproducer section entirely.

### Fingerprint

<fingerprint> is a hidden block of text that lets future agents disambiguate future bugs they see against existing reported bugs:

```
properties:
  * <failing antithesis property names, one bullet each>
related_code:
  * <function involved in the failure, one bullet each, as path/from/repo/root::function, or path::Type::method. No line numbers.>
root_cause: <a few short sentences explaining the bug's root cause, enough to tell whether another failure is the same bug, but not more>
```

### Duplicates

Before reporting, search the destination for duplicate issues. The fingerprints are helpful here to disambiguate different issues and decide if the bug is a duplicate of any of them.

If the bug already exists, edit the issue to add a bullet point to its <duplicate_details>.

<duplicate_details> has the following format:

```
## Other Antithesis runs

<details><summary>Details</summary>
<p>

* `run id`

</p>
</details>
```

and is only present if there are any duplicates, so you may have to create it. Do not include any content in each bullet other than `run id`.

## Before you finish

Check your report against these:

- Every property status you report came from the properties data, not inference.
- Conclusions about failures with moments rest on log evidence, with specific log lines, timestamps and property names.
- The report separates what the run shows from what you interpret.
- A claim that a binary isn't instrumented follows `references/triage/instrumentation.md` and doesn't rest only on the deduplicated examples of `Software was instrumented`.
