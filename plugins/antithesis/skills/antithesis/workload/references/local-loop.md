# Iterating locally

Coverage and the SDK assertions show what the workload reached. Measure both locally, so each round of fixes takes minutes.

## 1. Build the system for coverage

Antithesis instrumentation records nothing outside Antithesis, so use the language's own coverage. Add a `docker-compose.coverage.yaml` that overrides each system service with a coverage Dockerfile and a shared `coverage` volume, one directory per service.

**Rust**
- In the build stage, run `rustup component add llvm-tools-preview` and build with `RUSTFLAGS="-Cinstrument-coverage -Cllvm-args=-runtime-counter-relocation"`. This overrides the Antithesis flags in `.cargo/config.toml`.
- Set `LLVM_PROFILE_FILE=/coverage/<service>/%c%m.profraw`. `%c` writes counters as they change, so the data survives a killed process. `%m` merges restarts into one file. Don't use `%p`: every container's process is PID 1, so a restart overwrites the data.

**Go**
- Build with `go build -cover -covermode=atomic -coverpkg=./...`. Set `GOCOVERDIR=/coverage/<service>`.
- Go writes counters only when `main` returns or the program calls `os.Exit`.

**Other languages**
- Use the standard tool: clang's `-fprofile-instr-generate -fcoverage-mapping` for C and C++ (same `LLVM_PROFILE_FILE` rules as Rust), JaCoCo for Java, `c8` for Node, `coverage.py` for Python.
- Check that the data survives `docker compose stop` and a container restart before trusting it.

## 2. Run the workload

Start the coverage build from scratch, run the action generator for a few minutes with the SDK's local output on, then stop the system:

```bash
C="docker compose -f docker-compose.yaml -f docker-compose.coverage.yaml"
$C down -v && $C up -d --build --wait
$C exec -e ANTITHESIS_SDK_LOCAL_OUTPUT=/tmp/sdk.jsonl workload timeout 300 action-generator
$C cp workload:/tmp/sdk.jsonl .
$C stop
```

## 3. Check the behavior

- **Actions.** Every action reaches its intended outcome sometimes, and none is routinely slow.
- **Properties.** Only sometimes assertions that need faults should fail. This lists every property that would fail in a run:
  ```bash
  jq -rs '[.[].antithesis_assert // empty] | group_by(.message)[]
    | select(if .[0].assert_type == "sometimes" then any(.hit and .condition) | not
             else any(.hit and (.condition | not)) or (.[0].must_hit and (any(.hit) | not)) end)
    | "\(.[0].display_type)\t\(.[0].message)"' sdk.jsonl
  ```
  When a property fails, read the system's code for that request. If the system breaks a promise, keep the property: it found a real bug. If the property demands more than the system promises, fix the property.
- **Quiet periods.** One forced on passes against the healthy system.
- **Shutdown.** `docker stop` on a system container shuts its process down rather than killing it.

## 4. Read the coverage

After the system stops, generate the reports from the data in the `coverage` volume. They need the coverage binary, its sources and the coverage tools. The runtime image has only the binary, so use the coverage Dockerfile's build stage, which has all three:

```bash
docker build -f <service>/Dockerfile.coverage --target build -t <service>-coverage-build .
docker run --rm -v <project>_coverage:/coverage <service>-coverage-build sh -c '<report commands below>'
```

`<project>` is the Compose project name, by default the directory name.

**Rust:**

```bash
bin=$(rustc --print sysroot)/lib/rustlib/x86_64-unknown-linux-gnu/bin
$bin/llvm-profdata merge -sparse /coverage/<service>/*.profraw -o /tmp/m.profdata
$bin/llvm-cov report --show-functions -Xdemangler=c++filt -instr-profile=/tmp/m.profdata <binary> src/
```

Pass the system's own source directories, so dependencies stay out. To list only the functions that never ran, replace the last line with:

```bash
$bin/llvm-cov export -format=text -instr-profile=/tmp/m.profdata <binary> src/ |
  python3 -c 'import json,sys; [print(f["name"]) for f in json.load(sys.stdin)["data"][0]["functions"] if f["count"]==0]' | c++filt
```

**Go**, in the module directory: `go tool covdata textfmt -i=/coverage/<service> -o /tmp/cov.txt && go tool cover -func=/tmp/cov.txt`.

Compare the reports against the source tree. Look for modules with no reached file, files with many functions that never ran, and large branches that never ran in central functions. Error-handling and recovery code will look unreached without faults; leave it to the Antithesis runs.

## 5. Close the gaps

- **A request the workload never sends:** add it to the action generator.
- **A rare condition** (an error path or race): act only if a different action mix would make it common.

Repeat until the remaining gaps are rare conditions or out of scope.
