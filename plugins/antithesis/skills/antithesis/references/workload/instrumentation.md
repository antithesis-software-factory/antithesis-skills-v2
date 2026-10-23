# Instrumenting the system under test

Build every SUT binary with Antithesis coverage instrumentation. It's required: without it, Antithesis can't guide exploration or pause threads, and the run has no coverage data.

## Rules

- Instrument the binaries that run in the SUT's containers, and make sure the runtime image runs the instrumented build. The client library is code under test too: instrument the client program with it.
- Put symbol files in `/symbols` of the runtime image where the language needs them, or artifacts in `/opt/antithesis/catalog/` for languages Antithesis instruments itself.
- Add instrumentation tools and flags in the Antithesis Dockerfiles, not the project's own build.
- If a component can't be instrumented, instrument the rest.

The action generator doesn't need instrumentation. Its code isn't under test.

## By language

Follow the language's page in the Antithesis docs, including its validation step. The overview, with the list of supported languages, is `snouty docs show /docs/product/writing_tests/instrumentation/coverage_instrumentation/`.

- **Go:** `/docs/reference/sdk/go/instrumentation/`
- **Rust:** `/docs/reference/sdk/rust/instrumentation/`
- **C and C++:** `/docs/reference/sdk/cpp/instrumentation/`
- **Java:** `/docs/reference/sdk/java/how_to_build_with_sdk/`
- **.NET:** `/docs/reference/sdk/dotnet/instrumentation/`
- **JavaScript (Node):** `/docs/reference/sdk/javascript_sdk/`
- **Python:** `/docs/reference/sdk/python/`
- **Other languages compiled through LLVM:** `/docs/reference/sdk/fallback/`

If the project builds with link-time optimization, use the page's link-time instructions. Compile-time instrumentation doesn't survive LTO.

## Checking it

- Locally: the language's check passes, and the runtime image has the expected `/symbols` or catalog contents.
- In a run: `Software was instrumented` and `Every instrumented module has symbols` pass. If not, fix the build before reading other results.
