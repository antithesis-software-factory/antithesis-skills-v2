# Who talks to the system

Run the system's real client library in client containers, faulted like the servers, and have the action generator tell them what to do. Client libraries often hold part of the protocol: retry numbering, sessions, leases, multi-step commits, cached routing, batching. The servers' recovery code for these only runs when a client fails partway through, and a hand-written client never runs the library's own retry and recovery code.

Skip client containers only when the system has no client library, or its client keeps nothing between requests. Then the action generator sends requests to the servers itself, as in the example in `references/workload/action-generator.md`.

## The client program

A small long-running program in the library's language holds one client and exposes its operations to the action generator, for example over HTTP.

- **Configure the library as production users would.** Randomness and checks belong in the action generator.
- **Serve requests concurrently on one shared client**, so batching and pipelining happen. The action generator tags each operation with an ID. Send every completion to the action generator as an event, `{boot, op_id, outcome}`, even if the request has stopped waiting. Put the boot identifier in every response too.
- **Run it as the container's main process with `restart: on-failure`**, so a kill brings back a fresh client. Run several client containers, since many bugs sit between clients.
- **Forward its logs** like the servers' (`references/workload/system-logs.md`). Many interesting states sit inside one library call, and sometimes assertions on the logs are the only way to see them.

## Talking to the servers directly

A direct connection to the servers is a client that never fails. Use one only for read-only checks in quiet periods and for admin or metadata queries, never for actions.
