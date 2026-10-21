# Mapping what the workload must reach

A workload can only find bugs in code it makes the system run. Most of what a weak workload misses isn't bad luck. It's requests it never sends and features the deployment never turns on. Map the system first, then decide what to cover.

## 1. List the system's whole surface

Read the code that receives requests, not just the docs: the API definitions (protobuf, OpenAPI, route tables, command parsers, wire-protocol handlers) and the handlers behind them. Write down:

- **Every operation** a client can send.
- **Every field, flag, header or mode that sends a handler down a different branch.** One endpoint often hides several code paths. For example, a queue's ack modes and redelivery, a database's isolation levels and DDL, an HTTP service's auth, content types and bulk forms, or a storage API's conditional, ranged and streaming variants. Read each handler and note which inputs it branches on.
- **Background work the system does on its own**, such as cleanup, expiry, replication catch-up, rebalancing, leader election, retries, scheduled jobs and periodic flushes.
- **Configuration that changes behavior**, especially features that are **off by default** because a real client or operator is expected to turn them on or drive them. If a production deployment enables a feature, or a real client triggers it, the test must too.
- **Who the real clients are.** If the system mostly serves one kind of client (a framework, an SDK, another service), the requests that client sends are what matters most. Read how it calls the system, and prefer those shapes over a generic use of the API.

Prioritize by where the logic is. Operations real clients depend on, and code with the most branching (storage, replication, concurrency control), come first. A large API can't be covered evenly, so cover those deeply before adding rarely used operations.

## 2. Decide coverage, item by item

For every item on the list, either:
- name the action or setup that exercises it, or
- mark it out of scope and say why: for example, it needs a different deployment mode or an optional component this harness doesn't run, or it's a stub that only returns "not supported".

Write the table into the harness README. Put in scope what real clients depend on and where the logic is, as prioritized above. For a large API, say which parts you left for later, and why.

## 3. Design the deployment to reach it

- **Multiple nodes or replicas** if the system supports them, so replication, coordination and failover code runs.
- **Turn on background features** that are off by default, or drive them from the workload the way a real client would. Prefer both when both exist: they race each other.
- **Vary configuration between nodes** where the system documents that a setting may differ per node (intervals, batch sizes, pool sizes, cache sizes), so each node takes different code paths. Never vary settings that must match across a cluster, such as consensus timeouts or protocol versions.
- **Let failures be real.** The system's process must receive the signals Antithesis sends. Run it as the container's main process (`exec` it from any entrypoint script) or under an init that forwards signals. Don't restart it in a shell loop. Let the process exit, and give the system's containers `restart: on-failure` in `docker-compose.yaml`, so crashes are visible as container events and shutdown code runs.
