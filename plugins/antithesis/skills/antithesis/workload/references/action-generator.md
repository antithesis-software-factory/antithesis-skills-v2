# The action generator

The action generator is a hegel concurrent state machine run from `#[hegel::main]`. Each `#[rule]` is one action: it draws its inputs, sends one request to the system, and checks the response against the model. Hegel runs the rules on several worker threads at once, so the system sees concurrent clients.

Hegel runs the workers in rounds. Each worker runs a few rules, and the next round starts only when every worker is done. One slow rule therefore stalls every worker, not just its own.

## Example

This drives a replicated key-value store with `PUT /k/<key>` and `GET /k/<key>` on any of its nodes. The store promises linearizable reads, so the check accepts exactly the values a linearizable store could return. Copy the structure, but write the model and checks for **your system's documented guarantees**. A system with weaker guarantees, such as follower reads or eventual consistency, needs a looser check. Otherwise every correct response that a weaker system is allowed to give becomes a false failure.

`Cargo.toml`:

```toml
[dependencies]
hegeltest = { version = "0.50", features = ["static-engine"] }
antithesis_sdk = "0.3"
serde_json = "1"
ureq = "2"
```

`src/main.rs`:

```rust
use antithesis_sdk::prelude::*;
use hegel::TestCase;
use hegel::generators as gs;
use serde_json::json;
use std::sync::atomic::{AtomicU32, Ordering};
use std::sync::Mutex;
use std::time::{Duration, Instant};

const KEYS: usize = 8;
const TIMEOUT: Duration = Duration::from_millis(1000);

/// One write the workload sent. `end` is when the system acknowledged it, or `None`
/// if we never learned the outcome: such a write may take effect at any later time.
struct Write {
    value: u32,
    start: Instant,
    end: Option<Instant>,
}

struct Workload {
    nodes: Vec<String>,
    http: ureq::Agent,
    /// Every write sent to each key. Locks are held only to update or check the
    /// record, never during a request, so requests for the same key overlap.
    writes: Vec<Mutex<Vec<Write>>>,
    next_value: AtomicU32,
}

/// What a read that ran from `start` to `end` may return, given the writes to its key:
/// a value whose write began before the read ended and wasn't certainly overwritten
/// before the read began, or nothing if no write had certainly finished by then.
fn acceptable(got: Option<u32>, start: Instant, end: Instant, writes: &[Write]) -> bool {
    let overwritten_before_read = |w: &Write| {
        let Some(w_end) = w.end else { return false };
        writes.iter().any(|o| o.start > w_end && o.end.is_some_and(|e| e < start))
    };
    match got {
        None => !writes.iter().any(|w| w.end.is_some_and(|e| e < start)),
        Some(v) => writes.iter().any(|w| w.value == v && w.start < end && !overwritten_before_read(w)),
    }
}

/// Drops writes that no read can return any more: those certainly overwritten by a write
/// that finished well over a request timeout ago, so no read still in flight began before
/// it. Keeps the check cheap in long runs. Unacknowledged writes are kept, since they may
/// still take effect.
fn prune(writes: &mut Vec<Write>) {
    let now = Instant::now();
    // Starts of writes that finished long enough ago (twice the timeout, for margin).
    let settled: Vec<Instant> = writes
        .iter()
        .filter(|w| w.end.is_some_and(|e| now.duration_since(e) > 2 * TIMEOUT))
        .map(|w| w.start)
        .collect();
    writes.retain(|w| match w.end {
        None => true,
        Some(e) => !settled.iter().any(|o_start| *o_start > e),
    });
}

impl Workload {
    fn put(&self, node: usize, key: usize, value: u32) -> Result<(), ureq::Error> {
        self.http.put(&format!("{}/k/{key}", self.nodes[node])).send_string(&value.to_string()).map(|_| ())
    }

    fn get(&self, node: usize, key: usize) -> Result<Option<u32>, ureq::Error> {
        match self.http.get(&format!("{}/k/{key}", self.nodes[node])).call() {
            Ok(resp) => Ok(Some(resp.into_string()?.trim().parse().expect("non-numeric value"))),
            Err(ureq::Error::Status(404, _)) => Ok(None),
            Err(e) => Err(e),
        }
    }
}

#[hegel::concurrent_state_machine]
impl Workload {
    #[rule(group = "ops", weight = 2.0)]
    fn put_value(&self, tc: TestCase) {
        let node = tc.draw(gs::integers::<usize>().max_value(self.nodes.len() - 1));
        let key = tc.draw(gs::integers::<usize>().max_value(KEYS - 1));
        // Unique values make every read attributable to exactly one write.
        let value = self.next_value.fetch_add(1, Ordering::Relaxed);
        self.writes[key].lock().unwrap().push(Write { value, start: Instant::now(), end: None });
        // A failed write may still be applied later, so it keeps `end: None`.
        if self.put(node, key, value).is_ok() {
            let end = Instant::now();
            let mut w = self.writes[key].lock().unwrap();
            // Find the record by its value: pruning may have moved it.
            if let Some(rec) = w.iter_mut().find(|r| r.value == value) {
                rec.end = Some(end);
            }
            prune(&mut w);
        }
    }

    #[rule(group = "ops", weight = 3.0)]
    fn get_value(&self, tc: TestCase) {
        let node = tc.draw(gs::integers::<usize>().max_value(self.nodes.len() - 1));
        let key = tc.draw(gs::integers::<usize>().max_value(KEYS - 1));
        let start = Instant::now();
        // A failed read tells us nothing.
        let Ok(got) = self.get(node, key) else { return };
        let end = Instant::now();
        let writes = self.writes[key].lock().unwrap();
        assert_always!(
            acceptable(got, start, end, &writes),
            "a read returns the latest acknowledged write or one that overlapped it",
            &json!({"node": node, "key": key, "got": got})
        );
        assert_sometimes!(got.is_some(), "read finds a written key", &json!({"key": key}));
        assert_sometimes!(
            writes.iter().any(|w| w.start < end && w.end.map_or(true, |e| e > start)),
            "a read overlaps a write to the same key",
            &json!({"key": key})
        );
    }
}

#[hegel::main(phases = [hegel::Phase::Generate])]
fn main(tc: TestCase) {
    antithesis_init();
    let nodes = std::env::var("KV_URLS")
        .unwrap_or_else(|_| "http://kv1:8000,http://kv2:8000".to_string())
        .split(',')
        .map(str::to_string)
        .collect();
    let workload = Workload {
        nodes,
        // Short, so a request to an unreachable node costs a moment, not a stalled round.
        http: ureq::AgentBuilder::new().timeout(TIMEOUT).build(),
        writes: (0..KEYS).map(|_| Mutex::new(Vec::new())).collect(),
        next_value: AtomicU32::new(0),
    };
    hegel::stateful::machine(workload)
        .steps(i64::MAX)
        .max_concurrency(4)
        .run_concurrent(tc);
}
```

## Rules

**Run forever.** Always call `.steps(i64::MAX)`. Without it, the machine stops after 50 rounds and the binary exits. Don't use `tc.repeat` either: it draws a random iteration count and often exits after a few actions.

**Report properties with Antithesis SDK assertions, never by panicking.** Use `assert_always!` for things that must hold and `assert_sometimes!` for states the workload should reach at least once. Put the values you'd need to debug a failure in the details.

**Generate only.** Keep `phases = [hegel::Phase::Generate]` on `#[hegel::main]`. Shrinking re-runs the rules thousands of times against the live system.

**Cover the surface you mapped.** Every in-scope operation and option from the surface table (`references/surface.md`) gets an action, or a draw inside an action. Requests that differ only in an option (an ack mode, an isolation level, a batch form, a filter) usually take separate code paths in the system: draw the option, don't hard-code the common case.

**Let requests contend.** Bugs live where requests to the same data overlap. Don't serialize requests per key or per object to keep the model simple. Record when each request started and finished, and check responses against every outcome consistent with what overlapped them, as the example does. If the system assigns its own order to operations (a log position, a sequence number, a commit timestamp), use it: record each acknowledged operation at its position, and check what reads and subscriptions return against that history.

**Derive every property from what this system promises.** That means its documentation, its API contract, and what its real clients rely on. A system that implements another system's API often weakens some of its guarantees on purpose, for example how fresh a reported version is, or what an edge-case input does. Check each property against the system's own code path before relying on it. Where the system deliberately promises less than the API it imitates, check what it does promise.

**Treat unknown outcomes as unknown.** A request that timed out or lost its connection may or may not have taken effect, possibly much later. Keep it as possible, never as failed.

**Keep every rule fast.** Because of rounds, a rule that blocks stalls all workers.
- Use client timeouts that are short relative to the operation's normal latency, often around a second, so a request to an unreachable node fails fast. Operations that are slow by design (long polls, streams, schema changes, large batches) get their own longer timeout, or run as described below.
- Send each request to a randomly drawn node, so one node being down doesn't stop the workload.
- Don't sleep or poll inside a rule. A check that has to wait for something (expiry, convergence, delivery) should record what it expects and check it in a later rule, once the wait has passed.
- Long-lived interactions (subscriptions, streams, sessions, open transactions) live in the workload's state, not inside one rule. One rule opens them, other rules use, check or close them, and a background thread can drain a stream into a buffer that rules then check.

**Put liveness checks in an `eventually_` command, not in the action generator.** "Every node serves requests again once faults stop" belongs in a separate `eventually_` command (see `references/harness.md`). Antithesis stops faults and kills the action generator before running it, so it can't use the action generator's model. Checks that need the model stay in rules.

**Weight actions toward what's hard to reach.** Plain reads and writes run constantly anyway. Give options, background-feature triggers, and multi-step interactions enough weight that they happen many times in every timeline.

**Don't let one known bug drown out the rest.** If a bug fires constantly once found, for example an action that crashes the system every few seconds, record it in the harness README and turn down that action's weight so it still happens, but rarely. Otherwise the run spends most of its time on restarts of the same failure.

**Mark interesting states with sometimes assertions.** For each situation that exercises a distinct path (a conditional operation that loses a race, a request that fails over to another node, a retry that succeeds, a background job running during a request, a subscription that resumes after a disconnect), add an `assert_sometimes!`. They show which behaviors the run actually reached.

**Use `#[invariant(always_run)]`, not plain `#[invariant]`.** Hegel runs a plain invariant once on the initial state, and after that only with probability `1 / steps` per round, so with `.steps(i64::MAX)` it never runs again. An `always_run` invariant runs after every round, while no rules are running. Keep it cheap, because it runs often.

For the rest of hegel's API, like generators, composite types, and rule weights, see the hegel documentation.
