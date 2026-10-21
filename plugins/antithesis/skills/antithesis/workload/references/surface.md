# Mapping what the workload must reach

A workload can only find bugs in code it makes the system run. Most of what a weak workload misses isn't bad luck. It's requests it never sends and features the deployment never turns on. Map the system first, then decide what to cover.

## 1. List the system's whole surface

Read the code that receives requests, not just the docs: the API definitions (protobuf, OpenAPI, route tables, command parsers) and the handlers behind them. Write down:

- **Every operation** a client can send.
- **Every option and mode of each operation**: read-at-version, pagination and limits, count-only or metadata-only reads, conditional writes, batch forms, streaming forms, cancellation, TTLs and leases, consistency levels. These are often separate code paths behind one endpoint.
- **Background work the system does on its own**: compaction, garbage collection, expiry, snapshots, rebalancing, leader election, retries, periodic flushes.
- **Configuration that changes behavior**, especially features that are **off by default** because a real deployment's client or operator is expected to drive them. If production turns a feature on, or a real client triggers it, the test must too.
- **Who the real clients are.** If the system serves a particular client (an orchestrator, a framework, an SDK), the shapes of requests that client sends are what matters. Prefer them over a generic use of the API.

## 2. Decide coverage, item by item

For every item on the list, either:
- name the action or setup that exercises it, or
- mark it out of scope and say why: for example, it needs a different deployment (another storage backend, TLS), or it's a stub that only returns "not supported".

Write the table into the harness README. Anything a real client would use is in scope unless there's a concrete reason it can't run in this deployment.

## 3. Design the deployment to reach it

- **Multiple nodes or replicas** if the system supports them, so replication, coordination and failover code runs.
- **Turn on background features** that are off by default, or drive them from the workload the way a real client would. Prefer both when both exist: they race each other.
- **Vary configuration between nodes** where the system allows it (intervals, batch sizes, pool sizes, cache sizes), so each node takes different code paths.
- **Let failures be real.** The system's process must receive the signals Antithesis sends: run it as the container's main process (`exec` it from any entrypoint script) or under an init that forwards signals. Don't restart it in a shell loop. Let it exit and have the container restart, so crashes are visible as container events and shutdown code runs.
