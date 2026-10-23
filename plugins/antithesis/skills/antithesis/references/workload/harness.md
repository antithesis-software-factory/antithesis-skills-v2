# Wiring the action generator into Antithesis

## The workload container

Build the action generator into its own image and add it to `docker-compose.yaml` as a service named `workload`. Tell Antithesis to start the action generator through Test Composer:

```yaml
  workload:
    container_name: workload
    hostname: workload
    platform: linux/amd64
    build:
      context: .
      dockerfile: workload/Dockerfile
    image: workload:latest
    environment:
      KV_URLS: http://kv1:8000,http://kv2:8000
```

```dockerfile
FROM rust:1-bookworm AS build
WORKDIR /src
COPY workload/ .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=build /src/target/release/action-generator /usr/local/bin/
COPY workload/test/ /opt/antithesis/test/v1/
CMD ["sleep", "infinity"]
```

## The test template

The test template is a directory under `/opt/antithesis/test/v1/` in the workload container. It holds `singleton_driver_<name>`, which starts the action generator:

```
workload/test/main/singleton_driver_action_generator
```

```bash
#!/usr/bin/env bash
exec /usr/local/bin/action-generator
```

Make it executable.

## Client containers

If the real client library runs in client containers (`references/workload/clients.md`), add one service per client, built from the client program:

```yaml
  client1:
    container_name: client1
    hostname: client1
    platform: linux/amd64
    build:
      context: .
      dockerfile: client/Dockerfile
    restart: on-failure
    environment:
      KV_URLS: http://kv1:8000,http://kv2:8000
      LOG_FORWARD_ADDR: workload:7777
      OUTCOME_ADDR: workload:7778
```

Point the action generator at them instead of the servers, for example with `CLIENT_URLS: http://client1:8080,http://client2:8080`. `OUTCOME_ADDR` is where the client program sends its completion events.

The action generator may start before the client programs are reachable, and clients restart throughout a run anyway. Treat an unreachable client as a normal state the action generator handles, not a setup failure, and leave the clients out of `setup_complete`.

## Setup complete

Every timeline starts from the state at the moment something reports `setup_complete`, and Antithesis starts the action generator only after that. Report it from the system's side, once every node serves, from a check that retries rather than after a fixed sleep. With an Antithesis SDK, call its `setup_complete` function. Without one, append this line to `$ANTITHESIS_OUTPUT_DIR/sdk.jsonl`:

```json
{"antithesis_setup": {"status": "complete"}}
```

## Exclude workload container from faults

Exclude the workload container from faults with `advanced_test` parameters:

```bash
snouty launch --launcher advanced_test --config . --duration <minutes> \
  --param 'custom.network_fault_exclusion_patterns=^workload$' \
  --param custom.container_status_fault_exclusions=workload \
  --param custom.container_cpu_fault_exclusions=workload \
  ...
```

The container and CPU exclusions take exact container names. To exclude several containers from network faults, use `^a$|^b$`. Don't use parentheses (due to a bug).

Exclude only the workload container. Client containers get faulted like the servers: their failures are what they're there for.

Helper containers that only report setup don't need excluding.
