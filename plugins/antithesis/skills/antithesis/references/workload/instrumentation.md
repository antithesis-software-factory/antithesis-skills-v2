# Instrumenting the system under test

Build every SUT binary with Antithesis coverage instrumentation. It's required: without it, Antithesis can't guide exploration or pause threads, and the run has no coverage data.

## Rules

- Instrument the binaries that run in the SUT's containers, and make sure the runtime image runs the instrumented build. When the real client library runs in client containers, it's code under test too: instrument the client program with it.
- Put symbol files in `/symbols` of the runtime image where the language needs them, or artifacts in `/opt/antithesis/catalog/` for languages Antithesis instruments itself.
- Add instrumentation tools and flags in the Antithesis Dockerfiles, not the project's own build.
- If a component can't be instrumented, instrument the rest.

The action generator doesn't need instrumentation. Its code isn't under test.

## By language

Read the current page for the language with `snouty docs` first.

**Go** (`snouty docs show /docs/reference/sdk/go/instrumentation/`)
- Add the SDK module and install the instrumentor in the build stage: `go get github.com/antithesishq/antithesis-sdk-go@latest` and `go install github.com/antithesishq/antithesis-sdk-go/tools/antithesis-go-toolexec@latest`.
- Build with `-toolexec=antithesis-go-toolexec` added to the existing `go build`. The default mode does coverage instrumentation.
- Build with `CGO_ENABLED=1` and a C compiler; with cgo disabled, coverage is silently lost. Prefer glibc base images over Alpine.
- Copy the symbol files it writes into the runtime image: `COPY --from=build /symbols /symbols`.

**Rust** (`snouty docs show /docs/reference/sdk/rust/instrumentation/`)
- Add the `antithesis-instrumentation` crate and reference it once (`use antithesis_instrumentation as _;`).
- Build with the coverage flags, in `.cargo/config.toml`:
  ```toml
  [build]
  target = "x86_64-unknown-linux-gnu"

  [target.x86_64-unknown-linux-gnu]
  rustflags = [
      "-Ccodegen-units=1",
      "-Cpasses=sancov-module",
      "-Cllvm-args=-sanitizer-coverage-level=3",
      "-Cllvm-args=-sanitizer-coverage-trace-pc-guard",
      "-Clink-args=-Wl,--build-id",
  ]
  ```
- Put the unstripped binary or its debug info in `/symbols`.
- Check: `nm <binary> | grep antithesis_load_libvoidstar` prints a `T antithesis_load_libvoidstar` line.

**C and C++** (`snouty docs show /docs/reference/sdk/cpp/instrumentation/`)
- Include `antithesis_instrumentation.h` in exactly one translation unit.
- Compile with `-fsanitize-coverage=trace-pc-guard -g`, and link with `-Wl,--build-id`.
- Put the binary or its debug info in `/symbols`.
- Check with `nm <binary> | grep antithesis_load_libvoidstar`.

**Java, .NET, JavaScript (Node)**
- Antithesis instruments these itself. Put the built artifacts in `/opt/antithesis/catalog/`.
- Start Java apps with `java -cp …`, not `java -jar`.

**Python**
- No coverage instrumentation yet.

**Other languages compiled through LLVM**
- The C/Rust sanitizer-coverage flags may work, with the binary in `/symbols`. Not fully supported.

## Checking it

- Locally: the language's check passes, and the runtime image has the expected `/symbols` or catalog contents.
- In a run: `Software was instrumented` and `Every instrumented module has symbols` pass. If not, fix the build before reading other results.
