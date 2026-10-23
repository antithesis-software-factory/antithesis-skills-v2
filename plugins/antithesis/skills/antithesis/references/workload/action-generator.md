# The action generator

The action generator is a hegel concurrent state machine run from `#[hegel::main]`. Each `#[rule]` is one action: it draws its inputs, sends a request, and checks the response against the model. Hegel runs the rules on several worker threads, in rounds. The next round starts only when every worker is done, so one slow rule stalls them all.

## Example

This drives a replicated key-value store with no client library, with `PUT /k/<key>` and `GET /k/<key>` on any node, that promises linearizable reads.

Depend on `hegeltest` with `features = ["static-engine"]`, and on `antithesis_sdk`. Left out below: the HTTP client (`put`, `get`), reading the node list from `KV_URLS`, `draw_target` (a node and one of `KEYS` keys), and `KeyHistory`'s `sent`, `acked` and `overlaps` (a write was in flight during the read). `acked` finds its write by value, since pruning may have moved it.

```rust
/// Short, so a request to an unreachable node costs a moment, not a stalled round.
const TIMEOUT: Duration = Duration::from_millis(1000);
/// How long a write with an unknown outcome stays possible. Systems apply such writes
/// soon or never, and keeping them forever would let the check accept almost anything.
const AMBIGUITY_WINDOW: Duration = Duration::from_secs(60);

/// One write to a key. `acked` is when the system acknowledged it, or `None` if we never
/// learned the outcome, in which case it may still take effect.
struct Write { value: u32, sent: Instant, acked: Option<Instant> }

/// Every write sent to one key. A read that ran from `start` to `end` may return:
/// - a value, if its write was sent before the read ended and wasn't overwritten before
///   the read started;
/// - nothing, if no write had been acknowledged before the read started.
///
/// A write is overwritten before time `t` if another write was sent after it was
/// acknowledged, and was itself acknowledged before `t`.
#[derive(Default)]
struct KeyHistory { writes: Vec<Write> }

impl KeyHistory {
    fn may_return(&self, got: Option<u32>, start: Instant, end: Instant) -> bool {
        match got {
            None => !self.writes.iter().any(|w| w.acked.is_some_and(|a| a < start)),
            Some(v) => self.writes.iter().any(|w| w.value == v && w.sent < end && !self.overwritten_before(w, start)),
        }
    }

    fn overwritten_before(&self, w: &Write, t: Instant) -> bool {
        let Some(w_acked) = w.acked else { return false }; // it could still land later
        self.writes.iter().any(|o| o.sent > w_acked && o.acked.is_some_and(|a| a < t))
    }

    /// Forgets writes no read can return any more, so checks stay cheap in long runs: writes
    /// overwritten more than two timeouts ago (no read still in flight started before that),
    /// and unacknowledged writes older than `AMBIGUITY_WINDOW`.
    fn prune(&mut self) {
        let now = Instant::now();
        let Some(horizon) = now.checked_sub(2 * TIMEOUT) else { return };
        let keep: Vec<bool> = self.writes.iter().map(|w| match w.acked {
            None => now.duration_since(w.sent) < AMBIGUITY_WINDOW,
            Some(_) => !self.overwritten_before(w, horizon),
        }).collect();
        let mut keep = keep.into_iter();
        self.writes.retain(|_| keep.next().unwrap());
    }
}

struct Workload {
    nodes: Vec<String>,
    http: ureq::Agent,
    keys: Vec<Mutex<KeyHistory>>,
    next_value: AtomicU32,
}

// Locks are held only to update or check a key's history, never during a request, so
// requests to the same key overlap.
#[hegel::concurrent_state_machine]
impl Workload {
    #[rule(group = "ops", weight = 2.0)]
    fn put_value(&self, tc: TestCase) {
        let (node, key) = self.draw_target(&tc);
        // Unique values make every read attributable to exactly one write.
        let value = self.next_value.fetch_add(1, Ordering::Relaxed);
        self.keys[key].lock().unwrap().sent(value, Instant::now());
        let ok = self.put(node, key, value).is_ok();
        let acked_at = Instant::now();
        let mut history = self.keys[key].lock().unwrap();
        if ok {
            history.acked(value, acked_at);
        } // A failed write may still land, so it stays unacknowledged.
        history.prune();
    }

    #[rule(group = "ops", weight = 3.0)]
    fn get_value(&self, tc: TestCase) {
        let (node, key) = self.draw_target(&tc);
        let start = Instant::now();
        // `get` turns a 404 into `None`, and an unparseable body into a value no write produced.
        let Ok(got) = self.get(node, key) else { return }; // a failed read tells us nothing
        let end = Instant::now();
        let history = self.keys[key].lock().unwrap();
        assert_always!(
            history.may_return(got, start, end),
            "a read returns the latest acknowledged write or one that overlapped it",
            &json!({"node": node, "key": key, "got": got})
        );
        assert_sometimes!(got.is_some(), "read finds a written key", &json!({"key": key}));
        assert_sometimes!(history.overlaps(start, end), "a read overlaps a write to the same key", &json!({"key": key}));
    }
}

#[hegel::main(phases = [hegel::Phase::Generate])]
fn main(tc: TestCase) {
    antithesis_init();
    let workload = Workload { /* ... */ };
    hegel::stateful::machine(workload)
        .steps(i64::MAX)
        .max_concurrency(4)
        .run_concurrent(tc);
}
```

## Rules

- **Run forever.** Call `.steps(i64::MAX)`; the default stops after 50 rounds and the binary exits. Don't use `tc.repeat`.
- **Report properties with Antithesis SDK assertions**, never by panicking. Put the values you'd need to debug a failure in the details.
- **Generate only.** Keep `phases = [hegel::Phase::Generate]` on `#[hegel::main]`.
- **Cover the surface you mapped.** Draw options; don't hard-code the common case.
- **Let requests contend.** Don't serialize requests per key to keep the model simple. Check each response against every outcome consistent with what overlapped it, as the example does. If the system orders operations itself (log positions, sequence numbers), check against that history.
- **Derive properties from what this system promises:** its docs, its API contract, and what its real clients rely on. Check each against the system's code. Implementations often promise less than what they imitate.
- **Treat unknown outcomes as unknown.** A request that timed out or lost its connection may or may not have taken effect, now or later.

## Driving client containers

With client containers (`references/workload/clients.md`), the rules above still hold, with the action generator sending commands to a client program instead of requests to the servers. A few things change:

- **Each rule draws which client acts.** Bugs between clients need several of them acting on the same keys, locks or transactions.
- **Outcomes arrive as events.** Tag each operation with an ID and wait for the response only up to the usual short timeout. A background thread receives the client programs' completion events on their own UDP port, separate from the logs, and resolves each operation in the model as its event arrives, the way `references/workload/system-logs.md` checks log lines as they arrive. An operation without an event stays unknown until its client's boot changes or the ambiguity window passes. A dropped event leaves its operation unknown, which is safe.
- **Bursts are cheap.** Since outcomes arrive as events, a rule can send several operations to one client without waiting for their responses. That gets the library batching beyond what the worker count allows.
- **A changed boot identifier means the client restarted.** Everything outstanding on its old boot gets an unknown outcome, and the model forgets that client's sessions, locks and open transactions. Unknown includes what the client believed: a client that was paused, not restarted, may still think it holds a lock that has since expired.
- **An unreachable client is normal.** It may be paused or mid-restart. Treat commands to it as unknown, and keep driving the other clients.
- **Read-only checks may go to the servers directly**, as described in `references/workload/clients.md`. Actions never do.

**Keep rules relatively fast:**
- Use short timeouts. Give operations that are slow by design their own.
- Don't sleep or poll. Record what you expect, and check it in a later rule. The one exception is a quiet period (below), which waits for recovery on purpose.
- Keep long-lived interactions (streams, sessions, transactions) in the workload's state, across rules.
- Run blocking operations (watches, long-polls, blocking pops) on a background thread, record the result in the workload's state, and check it in a later rule. Bounded retries that mirror a real client's are fine inside a rule.

**Check eventual properties only in quiet periods** (below). Properties that hold only once faults stop, like every node serving again or acknowledged writes being readable, fail falsely during faults.

**Weight actions toward what's hard to reach**, so options and rare interactions happen many times in every timeline.

**Use `#[invariant(always_run)]`, not plain `#[invariant]`.** With `.steps(i64::MAX)`, a plain invariant effectively never runs. An `always_run` invariant runs after every round, while no rules run, and every round waits for it.

## Choosing assertions

- **`assert_always!`**: a property that should always be true.
- **`assert_always_or_unreachable!`**: a property that should always be true, but is rare enough that it may not get hit. This is the same as `assert_always!` but does not fail if it doesn't get hit. Only use when you would otherwise use `assert_always!` but do not want false positive errors from not reaching the code.
- **`assert_sometimes!`**: a condition that should be true at least once per test run. Sometimes assertions are useful for checking that tests exercise meaningful scenarios, not just particular pieces of code. Examples:
    - A leader election is triggered
    - A transaction aborts
    - A rate limiter rejects a request
    - A search returns no results
    - A cache entry expires and is refilled
    - A read returns a value written by another client
    - A compaction runs while reads are in flight
    - Two clients edit the same document at the same time

  If both scenarios A and ~A (A game is sometimes won and sometimes lost) are possible, add sometimes assertions for both.

- **`assert_reachable!`**: equivalent to `assert_sometimes!(true, ...)`.
- **`assert_unreachable!`**: something that should never happen.
- **Numeric and boolean forms** (`assert_sometimes_greater_than!`, `assert_sometimes_all!`, `assert_always_some!`, and so on): any always or sometimes condition that's a comparison or a combination. They tell Antithesis how close the condition came and guide its search.

Each message is one property in Antithesis, and assertions that share a message merge into it. Keep messages fixed and put changing values in the details.

## Choosing how often things happen

- **Groups control overlap, not frequency.** Each round hegel picks one group, and every worker runs only its rules. Use separate groups only for rules that must never overlap. Groups can't be weighted.
- **Weights are a hint** for the mix within a group, not a guarantee.
- **Explicit draws set exact probabilities.** Use `tc.draw(gs::weighted_booleans(p))`, or a phase in the workload's state, when a rate matters.

Put a rare action in the main group behind a rare draw, not in its own group, which would get as many rounds as any other. If it must not overlap other operations, run it from the `always_run` invariant behind a rare draw, as the quiet period does.

## Quiet periods

A quiet period pauses faults so the workload can check what should hold once the system recovers. Run the whole quiet period inside one call of an `#[invariant(always_run)]`. No rule runs until it returns, so its checks see a quiet system, and it can wait for recovery there. A per-round `weighted_booleans` draw starts quiet periods at a rate you choose. Pause faults with `ANTITHESIS_STOP_FAULTS <seconds>` for longer than the whole check takes; faults resume on their own afterwards. For the example above:

```rust
/// Chance per round of starting a quiet period. Measure rounds per second locally,
/// then pick this for roughly one quiet period every few minutes of a timeline.
const P_QUIET: f64 = 1.0 / 3000.0;
const QUIET: Duration = Duration::from_secs(60);    // how long faults stay paused
const DEADLINE: Duration = Duration::from_secs(45); // every node must serve by then

#[invariant(always_run)]
fn quiet_period(&self, tc: TestCase) {
    if !tc.draw(gs::weighted_booleans(P_QUIET)) {
        return;
    }
    if let Some(bin) = std::env::var_os("ANTITHESIS_STOP_FAULTS") {
        let _ = std::process::Command::new(bin).arg(QUIET.as_secs().to_string()).status();
    }
    // Restarted containers take a while to come back. Retry each node until it serves,
    // or until DEADLINE. With client containers, wait for them too.
    let healthy: Vec<bool> = self.wait_for_nodes(DEADLINE);
    let recovered = !healthy.contains(&false);
    assert_always!(recovered, "every node serves requests again once faults stop", &json!({"healthy": healthy}));
    if !recovered {
        return;
    }
    // Every node serves and no rule is running: check the model.
    for key in 0..KEYS {
        let start = Instant::now();
        let Ok(got) = self.get(0, key) else { return }; // a failed read tells us nothing
        let end = Instant::now();
        assert_always!(
            self.keys[key].lock().unwrap().may_return(got, start, end),
            "acknowledged writes survive once faults stop",
            &json!({"key": key, "got": got})
        );
    }
    assert_reachable!("the system recovered after faults stopped", &json!({}));
}
```

Clock faults continue during a pause, so allow for clock jumps. To test a quiet period locally, set the probability to 1.

For the rest of hegel's API, see the hegel documentation at https://hegel.dev/ (rust at https://docs.rs/hegeltest/latest/hegel/).
