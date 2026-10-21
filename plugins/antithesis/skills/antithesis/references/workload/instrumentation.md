# Instrumenting the system under test

Build every SUT binary with Antithesis coverage instrumentation. Coverage tells Antithesis which code each timeline reaches, which steers its exploration and enables thread pausing. It also produces the coverage data in the run's results. An uninstrumented SUT runs fine but is explored much less effectively, so instrumentation is required, not optional.

## Rules

- Instrument the binaries that actually run in the SUT's containers, not a separate local build.
- Make each SUT service's runtime image include its symbol files where its language needs them (`/symbols`), or its artifacts in `/opt/antithesis/catalog/` for languages Antithesis instruments itself. These go in the image that contains the instrumented software.
- Add instrumentation-only tools and flags in the Antithesis Dockerfiles, not the project's own build.
- Ensure the runtime image runs the instrumented artifact, not an uninstrumented build output.
- If a component truly can't be instrumented, say why in the harness's README and instrument the rest.

The action generator doesn't need instrumentation. It's the system that Antithesis should explore.

## By language

Before applying these, read the current Antithesis page for the language with `snouty docs`.

**Go** (`snouty docs show /docs/reference/sdk/go/instrumentation/`)
- Add the SDK module and install the instrumentor in the build stage: `go get github.com/antithesishq/antithesis-sdk-go@latest` and `go install github.com/antithesishq/antithesis-sdk-go/tools/antithesis-go-toolexec@latest`.
- Build with `-toolexec=antithesis-go-toolexec` added to the existing `go build`. The default mode does coverage instrumentation.
- Build with `CGO_ENABLED=1` and a C compiler in the build image. With cgo disabled, coverage is silently lost. Never set `CGO_ENABLED=0` on the Antithesis build path, even if the project normally does. Prefer glibc base images over Alpine.
- The instrumentor writes symbol files to `/symbols` in the build stage. Copy that directory into the runtime image: `COPY --from=build /symbols /symbols`.

**Rust** (`snouty docs show /docs/reference/sdk/rust/instrumentation/`)
- Add the `antithesis-instrumentation` crate, and reference it once in code (`use antithesis_instrumentation as _;`) so the linker keeps it.
- Build with the coverage flags, preferably in `.cargo/config.toml`. The crate and the flags only work together:
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
- Put the unstripped binary, or its debug info, in `/symbols` in the runtime image (a symlink is fine).
- Check with `nm <binary> | grep antithesis_load_libvoidstar`, which should print a line ending in `T antithesis_load_libvoidstar`.

**C and C++** (`snouty docs show /docs/reference/sdk/cpp/instrumentation/`)
- Include `antithesis_instrumentation.h` in exactly one translation unit.
- Compile with `-fsanitize-coverage=trace-pc-guard -g`, and link with `-Wl,--build-id`.
- Put the binary or its debug info in `/symbols`.
- Check with `nm <binary> | grep antithesis_load_libvoidstar`.

**Java, .NET, JavaScript (Node)**
- Antithesis instruments these itself. Put the built artifacts (`.jar`/`.war`, assemblies, or the JS tree with its `node_modules`) in `/opt/antithesis/catalog/`.
- For Java, start the app with an explicit classpath (`java -cp …`), not `java -jar`.

**Python**
- Antithesis has no coverage instrumentation for Python yet. Note this in the harness README.

**Other languages compiled through LLVM**
- Coverage may work with the same sanitizer-coverage flags as C and Rust, with the binary in `/symbols`. This path isn't fully supported.

## Checking it

- Locally: the language's check above passes, and the runtime image has the expected `/symbols` or `/opt/antithesis/catalog/` contents.
- In a run: the `Setup` group's `Software was instrumented` and `Every instrumented module has symbols` properties both pass. If either fails, fix the build before reading any other results.
