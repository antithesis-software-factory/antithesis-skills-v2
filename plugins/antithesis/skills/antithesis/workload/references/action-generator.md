# The action generator

The action generator is a hegel concurrent state machine run from `#[hegel::main]`. Each `#[rule]` is one action: it draws its inputs, sends one request to the system, and checks the response against the model. Hegel runs the rules on several worker threads at once, so the system sees concurrent clients.

## Example

This drives a key-value store with `PUT /k/<key>` and `GET /k/<key>`. Copy its structure and replace the actions, model, and properties with ones for your system.

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
use std::collections::BTreeSet;
use std::sync::Mutex;
use std::time::{Duration, Instant};

const KEYS: usize = 8;

struct Workload {
    url: String,
    http: ureq::Agent,
    /// For each key, every value a read could correctly return. `None` means
    /// the key is absent. A key's lock is held for the whole request, so the
    /// driver never sends two requests for the same key at once and the model
    /// stays exact. Requests for different keys still overlap.
    keys: Vec<Mutex<BTreeSet<Option<u32>>>>,
}

impl Workload {
    fn put(&self, key: usize, value: u32) -> Result<(), ureq::Error> {
        self.http
            .put(&format!("{}/k/{key}", self.url))
            .send_string(&value.to_string())
            .map(|_| ())
    }

    fn get(&self, key: usize) -> Result<Option<u32>, ureq::Error> {
        match self.http.get(&format!("{}/k/{key}", self.url)).call() {
            Ok(resp) => Ok(Some(resp.into_string()?.trim().parse().expect("non-numeric value"))),
            Err(ureq::Error::Status(404, _)) => Ok(None),
            Err(e) => Err(e),
        }
    }
}

#[hegel::concurrent_state_machine]
impl Workload {
    #[rule(group = "ops")]
    fn put_value(&self, tc: TestCase) {
        // Draw everything before taking a lock.
        let key = tc.draw(gs::integers::<usize>().max_value(KEYS - 1));
        let value = tc.draw(gs::integers::<u32>().max_value(1000));
        let mut possible = self.keys[key].lock().unwrap_or_else(|e| e.into_inner());
        match self.put(key, value) {
            Ok(()) => *possible = BTreeSet::from([Some(value)]),
            // A failed write may still have been applied.
            Err(_) => {
                possible.insert(Some(value));
            }
        }
    }

    #[rule(group = "ops")]
    fn get_value(&self, tc: TestCase) {
        let key = tc.draw(gs::integers::<usize>().max_value(KEYS - 1));
        let mut possible = self.keys[key].lock().unwrap_or_else(|e| e.into_inner());
        // A failed read tells us nothing.
        let Ok(got) = self.get(key) else { return };
        assert_always!(
            possible.contains(&got),
            "read returns a value that was written",
            &json!({"key": key, "got": got, "possible": *possible})
        );
        assert_sometimes!(got.is_some(), "read finds a written key", &json!({"key": key}));
        *possible = BTreeSet::from([got]);
    }

    #[rule(group = "check")]
    fn recovers_after_faults_stop(&self, tc: TestCase) {
        // Each round picks a group uniformly, so without this the check would run
        // in about half of all rounds and faults would rarely be on.
        if tc.draw(gs::integers::<u32>().max_value(49)) != 0 {
            return;
        }
        if let Some(stop_faults) = std::env::var_os("ANTITHESIS_STOP_FAULTS") {
            let _ = std::process::Command::new(stop_faults).arg("30").status();
        }
        let deadline = Instant::now() + Duration::from_secs(25);
        let mut healthy = false;
        while Instant::now() < deadline {
            if (0..KEYS).all(|key| self.get(key).is_ok()) {
                healthy = true;
                break;
            }
            std::thread::sleep(Duration::from_millis(500));
        }
        assert_always!(healthy, "every key is readable once faults stop", &json!({}));
    }
}

#[hegel::main(phases = [hegel::Phase::Generate])]
fn main(tc: TestCase) {
    antithesis_init();
    let workload = Workload {
        url: std::env::var("KV_URL").unwrap_or_else(|_| "http://kv:8000".to_string()),
        http: ureq::AgentBuilder::new().timeout(Duration::from_secs(10)).build(),
        keys: (0..KEYS).map(|_| Mutex::new(BTreeSet::from([None]))).collect(),
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

**Use `#[invariant(always_run)]`, not plain `#[invariant]`.** Hegel runs a plain invariant once on the initial state, and after that only with probability `1 / steps` per round, so with `.steps(i64::MAX)` it never runs again. An `always_run` invariant runs after every round, while no rules are running. That makes it a good place for checks that need a quiet system, but keep it cheap, because it runs often.

For more expensive checks, use a `#[rule]`. In the same group as the actions, it runs alongside other requests, so give it a low `weight` and make the check tolerate concurrent changes. If it needs a quiet system, put it in its own group. Hegel picks among groups uniformly each round, so gate the rule behind a rare draw, as the recovery check in the example does.

For the rest of hegel's API, like generators, composite types, and rule weights, see the hegel documentation.
