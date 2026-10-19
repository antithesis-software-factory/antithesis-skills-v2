# Wiring the action generator into Antithesis

## The workload container

Build the action generator into its own image and add it to `docker-compose.yaml` as a service named `workload`. Nothing else runs in it. Antithesis starts the action generator through Test Composer, so the container only needs to stay up:

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
      KV_URL: http://kv:8000
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

Pass the system's addresses in through environment variables, so the same binary also runs against a local `docker compose up`.

## The test template

The test template is a directory under `/opt/antithesis/test/v1/` in the workload container. It holds exactly one command, `singleton_driver_<name>`, which starts the action generator:

```
workload/test/main/singleton_driver_action_generator
```

```bash
#!/usr/bin/env bash
exec /usr/local/bin/action-generator
```

Make it executable. Don't add `parallel_driver_`, `serial_driver_`, `eventually_` or `finally_` commands. Concurrency and recovery checks belong in the action generator.

## Setup complete

Antithesis starts the action generator only after something reports `setup_complete`. Report it from the system's side, once the system can serve requests: from a startup script, a health-check container, or the system's own code. With an Antithesis SDK, call its `setup_complete` function. Without one, append this line to `$ANTITHESIS_OUTPUT_DIR/sdk.jsonl`:

```json
{"antithesis_setup": {"status": "complete"}}
```

## Keep faults away from the workload container

The action generator holds the model in memory and its requests should always arrive, so the workload container must be excluded from faults. Faults between the system's own containers stay on.

These settings are launch parameters, not part of `docker-compose.yaml`. On tenants whose `advanced_test` launcher supports them, launch with:

```bash
snouty launch --launcher advanced_test --config . --duration <minutes> \
  --param custom.network_fault_exclusion_patterns=workload \
  --param custom.container_fault_include_patterns=<system container names, comma-separated> \
  ...
```

- `custom.network_fault_exclusion_patterns` is a regex matched against container names. Every packet to or from a matching container is exempt from network faults.
- `custom.container_fault_include_patterns` is a comma-separated list of patterns. Kills, pauses, and CPU pressure only hit containers matching one of them. Leave `workload` out of it.
