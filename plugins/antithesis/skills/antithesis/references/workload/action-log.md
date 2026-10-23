<!-- DEV-ONLY: scripts/strip-dev-only deletes this file before release. -->
# Action log

The action generator must log every action it draws, including actions a rule decides to skip. The skill's evaluation tooling reads this log to measure throughput, outcomes and the system conditions each action ran under, so it's required.

Actions are buffered per worker and printed in batches, about once a second, to keep Antithesis's log volume low. Use the helper below as is.

## Format

Each worker prints one line per batch:

```
ANTITHESIS_ACTIONS {"w":2,"t0":81234.512,"tf":82236.004,"a":[[0.524,"put","c",0.52],[1.105,"get","c",0.02],[9.410,"update","s",0,"no known revision"]]}
```

| Field | Value |
|---|---|
| `w` | a small integer identifying the hegel worker thread |
| `t0` | when the batch started, in ms since the process started |
| `tf` | when the batch was printed, on the same clock |
| `a` | the batch's actions: `[end offset from t0 in ms, action, outcome, duration ms]`, plus a reason for skips |

`outcome` is one of:
- `c` (completed): the system gave a definite answer, yes or no. A conditional write whose condition fails is completed. Correctness is the assertions' job, not this field's.
- `e` (error): the system answered with an error.
- `n` (no answer): timeout, connection refused, or unreachable.
- `s` (skipped): the rule sent nothing. Give the reason.

If a rule sends several requests, its outcome is that of the first request that didn't complete.

## Helper

Copy this into the action generator. Call `log_action` or `log_skip` once at the end of every rule, on every path:

```rust
use std::cell::{Cell, RefCell};
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::OnceLock;
use std::time::Instant;

static START: OnceLock<Instant> = OnceLock::new();
static NEXT_WORKER: AtomicU64 = AtomicU64::new(0);
const BATCH_MS: f64 = 1000.0;

thread_local! {
    static WORKER: Cell<Option<u64>> = const { Cell::new(None) };
    static BATCH: RefCell<Option<(f64, Vec<serde_json::Value>)>> = const { RefCell::new(None) };
}

fn now_ms() -> f64 {
    START.get_or_init(Instant::now).elapsed().as_secs_f64() * 1000.0
}

fn worker_id() -> u64 {
    WORKER.with(|w| match w.get() {
        Some(id) => id,
        None => {
            let id = NEXT_WORKER.fetch_add(1, Ordering::Relaxed);
            w.set(Some(id));
            id
        }
    })
}

#[derive(Clone, Copy)]
enum ActionOutcome { Completed, Error, NoAnswer }

fn record(entry: impl FnOnce(f64) -> serde_json::Value) {
    let now = now_ms();
    BATCH.with(|b| {
        let mut b = b.borrow_mut();
        let (t0, actions) = b.get_or_insert_with(|| (now, Vec::new()));
        actions.push(entry(now - *t0));
        if now - *t0 >= BATCH_MS {
            let round = |x: f64| (x * 1000.0).round() / 1000.0;
            println!("ANTITHESIS_ACTIONS {}", serde_json::json!({
                "w": worker_id(), "t0": round(*t0), "tf": round(now), "a": actions,
            }));
            *b = None;
        }
    });
}

fn log_action(action: &str, started: Instant, outcome: ActionOutcome) {
    let duration = started.elapsed().as_secs_f64() * 1000.0;
    let code = match outcome {
        ActionOutcome::Completed => "c",
        ActionOutcome::Error => "e",
        ActionOutcome::NoAnswer => "n",
    };
    record(|end| serde_json::json!([(end * 1000.0).round() / 1000.0, action, code, (duration * 1000.0).round() / 1000.0]));
}

fn log_skip(action: &str, reason: &str) {
    record(|end| serde_json::json!([(end * 1000.0).round() / 1000.0, action, "s", 0, reason]));
}
```

Write the mapping from your client's errors to `Error` and `NoAnswer` once, as a function, and use it in every rule.
