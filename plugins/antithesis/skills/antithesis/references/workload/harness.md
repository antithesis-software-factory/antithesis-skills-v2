# Wiring the driver into Antithesis

## The driver container

Build the driver into its own image and add it to `docker-compose.yaml` as a service named `driver`. The driver is the container's main process, and the container waits for the snapshot gate (below):

```yaml
  driver:
    container_name: driver
    hostname: driver
    platform: linux/amd64
    build:
      context: .
      dockerfile: driver/Dockerfile
    image: driver:latest
    environment:
      KV_URLS: http://kv1:8000,http://kv2:8000
    depends_on:
      snapshot-gate:
        condition: service_completed_successfully
```

```dockerfile
FROM rust:1-bookworm AS build
WORKDIR /src
COPY driver/ .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=build /src/target/release/driver /usr/local/bin/
CMD ["/usr/local/bin/driver"]
```

## The snapshot gate

Every timeline starts from the root snapshot, which Antithesis takes some time after the system reports `setup_complete` (below). The driver must start after the snapshot, so that each timeline runs its own. A snapshot-gate container holds it back:

1. The gate waits for `/first_open_gate_done`.
2. The system reports `setup_complete`, and Antithesis takes the root snapshot.
3. At the start of each timeline, Antithesis runs the gate's `first_open_gate` test command, which creates `/first_open_gate_done`.
4. The gate exits, and Compose starts the driver.

The system's containers boot before the snapshot, so every timeline starts with the system already up.

```yaml
  snapshot-gate:
    container_name: snapshot-gate
    hostname: snapshot-gate
    platform: linux/amd64
    build:
      context: .
      dockerfile: snapshot-gate/Dockerfile
    image: snapshot-gate:latest
```

```dockerfile
FROM debian:bookworm-slim
COPY snapshot-gate/wait-for-snapshot /usr/local/bin/
COPY snapshot-gate/first_open_gate /opt/antithesis/test/v1/main/first_open_gate
CMD ["wait-for-snapshot"]
```

`snapshot-gate/wait-for-snapshot`:

```bash
#!/usr/bin/env bash
while [[ ! -e /first_open_gate_done ]]; do sleep 1; done
```

`snapshot-gate/first_open_gate`:

```bash
#!/usr/bin/env bash
touch /first_open_gate_done
```

Make both executable.

## Client containers

Add one service per client container (`references/workload/clients.md`), built from the client program:

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
      LOG_FORWARD_ADDR: driver:7777
      OUTCOME_ADDR: driver:7778
```

Point the driver at them instead of the servers, for example with `CLIENT_URLS: http://client1:8080,http://client2:8080`. `OUTCOME_ADDR` is where the client program sends its completion events.

The driver may start before the client programs are reachable, and clients restart throughout a run anyway. Treat an unreachable client as a normal state the driver handles, not a setup failure, and leave the clients out of `setup_complete`.

## Setup complete

Report `setup_complete` from the system's side, once every node serves, from a check that retries rather than after a fixed sleep. With an Antithesis SDK, call its `setup_complete` function. Without one, append this line to `$ANTITHESIS_OUTPUT_DIR/sdk.jsonl`:

```json
{"antithesis_setup": {"status": "complete"}}
```

## Exclude the driver container from faults

Exclude the driver container from faults with these `advanced_test` parameters. Launch every run of the workload with the `advanced_test` launcher and these parameters, following `references/launch/MODULE.md`:

```bash
--param 'custom.network_fault_exclusion_patterns=^driver$'
--param custom.container_status_fault_exclusions=driver
--param custom.container_cpu_fault_exclusions=driver
```

The container and CPU exclusions take exact container names. To exclude several containers from network faults, use `^a$|^b$`. Don't use parentheses (due to a bug).

Exclude only the driver container. Client containers get faulted like the servers: their failures are what they're there for.

The snapshot gate doesn't need excluding.
