# Asserting on the system's logs

The system's logs show what each node believes: who is leader, which term it's in, how far it has applied. A forwarder in each system container sends them to the workload, where a background thread in the action generator asserts properties over that state, and marks states across nodes worth exploring from.

Client containers (`references/workload/clients.md`) are part of the system too. Forward their logs the same way.

## 1. Find the log lines

Find the log lines that show the state your properties need, at any level. Turn on the level you need through the system's own configuration in `docker-compose.yaml`.

Prefer lines that state current values over lines that announce a change. The action generator starts after `setup_complete`, so it misses early lines.

## 2. Forward the logs

Add a forwarder, `workload/log-forward/`, to the harness: a small Rust binary that

- copies stdin to stdout, flushing each line;
- sends each line to `LOG_FORWARD_ADDR` over UDP, as `{"node": <$HOSTNAME>, "boot": <id>, "seq": <n>, "line": <text>}`;
- takes `boot` from the clock at startup, not the PID, which is the same on every container restart;
- counts `seq` from 0;
- never blocks or exits on network errors. It resolves the address once, retrying at most once a second until it does, and drops lines it can't send.

Run the system's command through it with `workload/forward-logs`, executable:

```bash
#!/usr/bin/env bash
exec "$@" 2> >(exec log-forward >&2)
```

If the system logs to stdout, use `exec "$@" > >(exec log-forward)`.

In each system service's Antithesis Dockerfile, build the forwarder statically, so it runs on any base image, and put the wrapper in front of the system's command:

```dockerfile
FROM rust:1-bookworm AS log-forward
RUN rustup target add x86_64-unknown-linux-musl
COPY workload/log-forward/ /src/
RUN cd /src && cargo build --release --target x86_64-unknown-linux-musl

# In the runtime stage:
COPY --from=log-forward /src/target/x86_64-unknown-linux-musl/release/log-forward /usr/local/bin/
COPY workload/forward-logs /usr/local/bin/
ENTRYPOINT ["forward-logs", "kv-server"]
```

- Setting `ENTRYPOINT` drops a `CMD` inherited from the base image. Restate it if there is one.
- The wrapper needs bash.

Set `LOG_FORWARD_ADDR=workload:7777` on each system service.

## 3. Parse and assert

In `main`, bind a UDP socket on port 7777 before starting the state machine, and parse lines on a background thread. Keep the node states behind an `Arc<Mutex<…>>` shared with the workload.

For the key-value store in `references/workload/action-generator.md`, if its nodes log `msg=raft-state term=3 role=leader commit=42`:

```rust
/// A node's state from one `raft-state` line, with the line's `boot` and `seq`.
struct NodeState { boot: u64, seq: u64, term: u64, role: String, commit: u64 }

#[derive(Default)]
struct NodeStates {
    latest: HashMap<String, NodeState>, // by node
    leaders: HashMap<u64, String>,      // the first leader seen in each term
}

impl NodeStates {
    fn record(&mut self, node: String, s: NodeState) {
        assert_reachable!("the workload reads a node's state from its logs", &json!({}));
        if s.role == "leader" {
            let first = self.leaders.entry(s.term).or_insert_with(|| node.clone());
            assert_always!(*first == node, "at most one leader per term",
                &json!({"term": s.term, "leaders": [first, &node]}));
        }
        if let Some(prev) = self.latest.get(&node).filter(|p| p.boot == s.boot) {
            if s.seq <= prev.seq {
                return; // arrived out of order
            }
            assert_always!(s.commit >= prev.commit, "a node's commit index never goes backwards",
                &json!({"node": node, "from": prev.commit, "to": s.commit}));
        }
        self.latest.insert(node, s);

        // `latest` mixes lines sent at different times: fine for sometimes assertions,
        // not for safety properties.
        let nodes: Vec<&NodeState> = self.latest.values().collect();
        assert_sometimes!(nodes.iter().any(|n| n.role == "candidate"), "a node stands for election", &json!({}));
        assert_sometimes!(nodes.iter().any(|n| n.term != nodes[0].term), "nodes disagree on the term", &json!({}));
        if let Some(l) = nodes.iter().filter(|n| n.role == "leader").max_by_key(|n| n.term) {
            let lag = nodes.iter().map(|n| l.commit.saturating_sub(n.commit)).max().unwrap_or(0);
            assert_sometimes_greater_than!(lag, 10, "a follower falls far behind the leader", &json!({}));
        }
    }
}
```

If nodes also log `msg=apply key=3 value=17`, check each applied value against the model as the line arrives. Share the action generator's `next_value` with the thread as an `Arc<AtomicU32>`. Values are drawn in order before they're sent, so every value the workload wrote is below it:

```rust
assert_always!(value < next_value.load(Ordering::Relaxed), "nodes apply only values the workload wrote",
    &json!({"node": node, "value": value}));
```

Check against `next_value`, not the key histories, which forget old writes that a lagging node may still apply. Parse a malformed value as one the workload never wrote, so the check reports it.

## Rules

- **Assert as lines arrive**, in the receiving thread, not in rules. A rule runs late and misses short-lived states.
- **Share state both ways.** Rules read the node states, for example to send reads to a lagging follower. Give the receiving thread the model too (behind an `Arc`), so properties that combine the logs with the model are checked when the line arrives, such as every committed value being one the workload wrote. Never hold a model lock and the node-state lock at once: copy what you need from one, release it, then take the other.
- **Don't depend on arrival order across nodes.** Lines from different nodes arrive in any order. Check safety over everything seen (one leader per term), or over one node's lines in `seq` order. The latest state of several nodes is a mix of times: use it only for sometimes assertions.
- **Mark states worth exploring from** with sometimes assertions, especially combinations across nodes: a leader change while a follower lags, a node rejoining with a stale term, nodes disagreeing on membership. Use the numeric forms when there's a distance to the state.
- **Assert that every kind of line you parse arrives**, with a reachable assertion per kind. A changed format or a disabled log level otherwise looks like a quiet system.
