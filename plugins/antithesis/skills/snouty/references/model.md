# Antithesis conceptual model

## Definitions

- **Run**: A full Antithesis fuzzing/test run, identified by a run ID (for example: `1d0b6331ec02e8268fee157346701f01-64-0`).
- **Moment**: A uniquely identified instant in a run. Identified by an (`input_hash`, `vtime`) tuple, where `input_hash` identifies the timeline, and `vtime` identifies the virtual time.[^alias]
- **Timeline**: The sequence of moments of one continuous simulated execution, in `vtime` order.
- **Event**: A record logged at a moment. An event can be output from the system (stdout or stderr), an Antithesis SDK assertion result, an Antithesis test command starting or finishing, or an Antithesis-injected fault. Not all moments have events.

Conceptually, a run is a tree, where each point on the tree is a moment, and each unique path on the tree from the root to a leaf is a timeline.

## Commands

- `snouty runs properties <run_id> --failing --detail`: see failing properties along with their moments.
- `snouty runs logs <run_id> <input_hash> <vtime>`: all events on the given moment's timeline, up to that moment.
  - add `--detail` for precise vtimes and more assertion details.
- `snouty runs search <run_id> '<query>'`: search for matching events across all timelines.
  - See `snouty runs search --help` for the syntax of `<query>`.

## Executing commands

You can execute an arbitrary bash command at a given moment using `snouty runs exec`. This is a powerful tool for exploration and debugging.

```bash
snouty runs exec <run_id> <input_hash> <vtime> '<bash_script>'
```

As part of its output, `snouty runs exec` prints an "end moment", which is the moment where `<bash_script>` finished executing. You can then use this moment like any other moment, including chaining together a new `snouty runs exec` command from that moment.

For a moment from a different run, pass that run as `--source-run-id`. The run you execute in first recreates the moment by replaying the simulation up to it (snouty calls this rewarming). This takes from seconds to a few minutes, longer for a later `vtime`.

- Use `--container <name>` to execute `<bash_script>` inside that container.
- Use `--events` to also print out all events that happen while the command executes.

You can combine `--events` with eg `sleep 30`, to see what the system does in those 30 seconds. A nice trick is to do this after performing some action, like killing a container, to see the result of that action on the system.

Note that the run `<run_id>` must currently be in progress to use `snouty runs exec`.

## Debugging runs

`snouty runs exec` only works on runs that are in progress. To execute commands at a moment from a finished run, start a debugging run at that moment with `snouty debug`:

```bash
run_id=$(snouty --json debug --run-id <run_id> --input-hash <input_hash> --vtime <vtime> | jq -r .runId)
```

A debugging run is a normal run, except it doesn't test anything. It just waits for you to execute commands in it.

To start, you can execute commands at the moment you passed to `snouty debug`, and at the end moment of any command you execute in the debugging run. If you want to execute at any other moment, use `--source-run-id`:

```bash
snouty runs exec $run_id <original_hash> <original_vtime> 'ps aux'
snouty runs exec $run_id <different_hash> <different_vtime> --source-run-id <run_id> 'ls /data'
```

Debugging runs can be used to explore moments from any run, not just the original `<run_id>`:

```bash
snouty runs exec $run_id <other_hash> <other_vtime> --source-run-id <other_run_id> 'ps aux'
```

It is cheaper and faster to reuse an existing debugging run to explore a moment from other runs than to start a new debugging run for each run.

## Known problems

### Debugging runs take a while to start

A new debugging run can't execute commands for about 10 minutes. `snouty runs wait $D --until running` returns several minutes before the run is actually ready. In the meantime, `snouty runs exec` fails with `404 Resource not found` or `Moment not warm`, even at the moment you passed to `snouty debug`, and snouty suggests adding `--source-run-id`. Ignore that suggestion. The run just isn't ready yet. Retry until it works:

```bash
snouty runs wait $D --until running
for i in $(seq 40); do snouty runs exec $D <input_hash> <vtime> 'true' && break; sleep 30; done
```

If it's still failing after 20 minutes, check `snouty runs show $D`.

### Cancel can report failure when it worked

`snouty runs cancel $D` can print `has already finished` and exit 1 even though the cancel worked. Check `snouty runs show $D`. If the status is no longer `in_progress`, the cancel worked.

[^alias]: Technically, a moment can be referred to by more than one (`input_hash`, `vtime`) pair: an `input_hash` from later on the same timeline, paired with the moment's `vtime`, also refers to it. But this is rarely useful in practice. Prefer the 'canonical' (`input_hash`, `vtime`) pair given by snouty.
