<!-- DEV-ONLY: scripts/strip-dev-only deletes this file before release. -->
# Action log

Every action the action generator draws must print one log line when it finishes. The skill's evaluation tooling reads these lines to measure throughput, outcomes and coverage of system conditions, so they're required.

## Format

One line on stdout per action, including actions the rule decided to skip:

```
ANTITHESIS_ACTION {"action":"put","worker":2,"outcome":"completed","duration_ms":6.8}
ANTITHESIS_ACTION {"action":"delete","worker":0,"outcome":"skipped","duration_ms":0.0,"reason":"revision unknown"}
```

| Field | Value |
|---|---|
| `action` | the rule's name |
| `worker` | a small integer identifying the hegel worker thread |
| `outcome` | `completed`, `error`, `no_answer` or `skipped` |
| `duration_ms` | time from the rule's start to its end |
| `reason` | required when `outcome` is `skipped`, otherwise omitted |

## Outcomes

- `completed`: the system gave a definite answer, yes or no. A conditional write whose condition fails is completed. Correctness is the assertions' job, not this field's.
- `error`: the system answered with an error.
- `no_answer`: timeout, connection refused, or unreachable.
- `skipped`: the rule sent nothing. Say why in `reason`.

If a rule sends several requests, its outcome is that of the first request that didn't complete.

## Helper

Copy this into the action generator and call it once at the end of every rule, on every path:

```rust
use std::cell::Cell;
use std::sync::atomic::{AtomicU64, Ordering};
use std::time::Instant;

static NEXT_WORKER: AtomicU64 = AtomicU64::new(0);
thread_local!(static WORKER: Cell<Option<u64>> = const { Cell::new(None) });

fn worker_id() -> u64 {
    WORKER.with(|w| *w.get().get_or_insert_with(|| NEXT_WORKER.fetch_add(1, Ordering::Relaxed)))
}

#[derive(Clone, Copy)]
enum ActionOutcome { Completed, Error, NoAnswer }

fn log_action(action: &str, started: Instant, outcome: ActionOutcome) {
    let outcome = match outcome {
        ActionOutcome::Completed => "completed",
        ActionOutcome::Error => "error",
        ActionOutcome::NoAnswer => "no_answer",
    };
    println!("ANTITHESIS_ACTION {}", serde_json::json!({
        "action": action, "worker": worker_id(), "outcome": outcome,
        "duration_ms": started.elapsed().as_secs_f64() * 1000.0,
    }));
}

fn log_skip(action: &str, reason: &str) {
    println!("ANTITHESIS_ACTION {}", serde_json::json!({
        "action": action, "worker": worker_id(), "outcome": "skipped",
        "duration_ms": 0.0, "reason": reason,
    }));
}
```

Write the mapping from your client's errors to `Error` and `NoAnswer` once, as a function, and use it in every rule.
