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

Pass the system's addresses in through environment variables, so the same binary also runs against a local `docker compose up`.

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

## Setup complete

`setup_complete` is the line between deploying the system and testing it. Reporting it says the system is now in the state every timeline should start from. Antithesis starts the action generator only after something reports it. Report it from the system's side, once the system can serve requests. With an Antithesis SDK, call its `setup_complete` function. Without one, append this line to `$ANTITHESIS_OUTPUT_DIR/sdk.jsonl`:

```json
{"antithesis_setup": {"status": "complete"}}
```

- Report it from a check that retries until every node serves, not after a fixed sleep.

## Exclude workload container from faults

The action generator holds the model in memory and its requests should always arrive, so the workload container must be excluded from faults. This is configured through `advanced_test` parameters:

```bash
snouty launch --launcher advanced_test --config . --duration <minutes> \
  --param 'custom.network_fault_exclusion_patterns=^workload$' \
  --param custom.container_status_fault_exclusions=workload \
  --param custom.container_cpu_fault_exclusions=workload \
  ...
```

The container and CPU exclusions take exact container names. To exclude several containers from network faults, use `^a$|^b$`. Don't use parentheses (due to a bug).

Helper containers that only report setup don't need excluding.
