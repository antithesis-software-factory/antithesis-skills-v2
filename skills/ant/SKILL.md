---
name: ant
description: Entry point to Antithesis. Use for anything involving Antithesis runs, properties, or workloads - figuring out why a run failed, debugging a property failure in the multiverse debugger, building or improving a workload, or importing and defining properties.
---

# Antithesis

Pick the module from the first argument, then read only that module's `MODULE.md`.

| Argument | Module | Use when |
|---|---|---|
| `triage` | [triage/MODULE.md](triage/MODULE.md) | A run has started or finished and you need to know what failed. |
| `debug` | [debug/MODULE.md](debug/MODULE.md) | A specific property failure needs investigating in the multiverse debugger. |
| `workload` | [workload/MODULE.md](workload/MODULE.md) | Building or iterating on a workload. |
| `properties` | [properties/MODULE.md](properties/MODULE.md) | Importing existing properties (`import`) or defining a new one (`define`). |

If there is no argument, infer the module from the request. If it is still ambiguous, ask.
