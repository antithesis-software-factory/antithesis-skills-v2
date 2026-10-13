# Triage an Antithesis run

Input: an Antithesis run ID. Set up snouty with the `snouty` skill. Use `SNOUTY_UNSTABLE_FEATURES=runs-search` when using snouty.

Read `antithesis-triage/antithesis-triage.md` and follow its "Triage a run" and "Investigate failed properties" workflows with the following changes:

- Don't ask the user for input. The tenant is in `ANTITHESIS_TENANT`.
- Find events with `snouty runs search -n 999`, not `snouty runs events`.
- Skip "Suggesting follow-up skills".
- Report as in "Report issues" below, not as `antithesis-triage.md` describes.
- Triage each failing property in its own subagent.

## Find failing properties

`snouty --json runs show <run_id>` gives the run's status.

* For a completed run: `snouty --json runs properties <run_id>` lists every property, one JSON object per line. Failing ones have `"status": "Failing"` and example moments.
* For a live run: `snouty runs properties` only returns results once the run completes. Instead, stream events as they happen:

    ```bash
    snouty --json runs search --follow -n 999 <run_id> 'filter(ev => ev.antithesis_assert?.hit
      && ev.antithesis_assert.condition === false && ev.antithesis_assert.assert_type != "sometimes")'
    ```

    - `--follow` replays past matches, then prints new ones.
    - `antithesis_assert.id` is the property name.
    - Triage each property as soon as its first failure appears. Don't wait for the run to complete.
    - `--follow` exits when the run finishes, or after 999 events. If `runs show` says the run is still in progress, reconnect and exclude properties already seen with `&& !["<id>", ...].includes(ev.antithesis_assert.id)`.
    - When the run completes, check `runs properties` for `sometimes` and `reachable` properties, which fail by never being hit and are not included in `--follow`.
    - If the run can't be found, wait 10 seconds and retry. There may be some delay between the run calling the webhook, and the run being visible in the API.

Pitfalls:
- `runs logs` needs an exact moment. Negative hashes need `--`.

## Report issues

Report one issue for each root cause among the failed Antithesis properties. Do not report multiple issues for the same root cause, even if multiple properties failed.

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
