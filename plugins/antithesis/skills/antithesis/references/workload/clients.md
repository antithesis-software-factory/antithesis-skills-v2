# Who talks to the system

The action generator runs without faults so it can always act and see everything. Whether it can also be the system's client depends on what the client does.

A simple client sends a request and waits, and all the state lives on the servers. A crashed or partitioned client looks to the servers like one that never sent the request, so faulting it finds nothing new. The action generator can be the client, as in the key-value example in `references/workload/action-generator.md`.

## Clients that take part in the protocol

Some client libraries hold part of the protocol: they number writes so servers can drop retried duplicates, hold sessions or leases, drive multi-step commits, route from cached cluster metadata, or batch writes. The servers rely on the client to finish what it started, and their recovery code only runs when a client fails partway through. In a Percolator-style transaction, a client that dies between committing the primary key and clearing the other locks leaves locks that later readers must resolve. A client paused past its lease expiry wakes up still believing it holds the lock, and fencing has to stop it. A client that never fails never reaches this code, and a hand-written client never runs the real library's retry and recovery logic, which is code under test even when only the servers are faulted.

For these systems, run the real client library in its own containers, faulted like the servers, and have the action generator tell them what to do. The workload's fault exclusion covers traffic in both directions, so it can keep driving a client partitioned from the servers, as the lease case needs.

Decide this while mapping the surface. Splitting costs a program in another language, more containers and more model state, so it's only worth it when the client takes part in the protocol. For in-between cases, like database drivers with pools and retries, ask whether the servers rely on anything the client remembers between requests.

## The client program

A small long-running program in the library's language holds a client and exposes its operations to the action generator, for example over HTTP. It should be invisible: its own retries or timeouts, or a configuration real users wouldn't run, change the client under test. Configure the library as the docs tell production users to, and keep randomness and checks in the action generator.

Serve requests concurrently on one shared client. Many client bugs live in batching and pipelining, which only happen when several operations are in flight on the same client, and concurrent requests from the action generator's workers provide that. So the action generator tags each operation with an ID, and the program reports every completion as an event, `{boot, op_id, outcome}`, sent to the action generator whether or not the request is still waiting. The response carries the outcome too when it arrives in time.

Make it the container's main process with `restart: on-failure`, so a kill brings back a fresh client like a real restart. Run several client containers, since many bugs sit between clients: one fencing another, conflicting transactions, a restarted client taking over from its old self.

## What the action generator can't see

Client state is now system state, visible only through responses, events and logs, which all stop when the client is faulted.

- **A restarted client loses everything it held.** Put a boot identifier in every response; everything outstanding from an old boot has an unknown outcome.
- **Client beliefs can be unknown too**, like whether it still thinks it holds a lock.
- **Interesting states often sit inside one library call.** The action generator can only make them likely. Forward client logs like the servers' (`references/workload/system-logs.md`) and mark these states with sometimes assertions from them.

Fault-injected kills rarely land mid-operation. If that matters, let the client program discard its client on command, without cleanup, so hegel controls how often a client walks away halfway through.

## Talking to the servers directly

Checking through a client is awkward: it must be up, its library may cache or retry in ways that blur what the cluster holds, and admin queries may not be in the library. But a direct connection is a client that never fails. Use it only for read-only checks in quiet periods and admin or metadata queries, never for actions.
