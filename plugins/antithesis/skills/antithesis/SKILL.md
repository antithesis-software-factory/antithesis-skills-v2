---
name: antithesis
description: Entry point to Antithesis. Use for anything involving Antithesis runs, properties, or workloads - figuring out why a run failed, debugging a property failure in the multiverse debugger, building or improving a workload, or importing and defining properties.
argument-hint: "[triage [monitor|summary|root-cause]|debug|workload|properties] [target]"
---

# Antithesis

Always read `references/antithesis/explainer.md` before starting.

Pick the module from the first argument, then read only that module's `MODULE.md`.

| Argument     | Module                            | Use when                                                                   |
| ------------ | --------------------------------- | -------------------------------------------------------------------------- |
| `triage`     | `references/triage/MODULE.md`     | A run has started or finished and you need to know what failed.            |
| `debug`      | `references/debug/MODULE.md`      | A specific property failure needs investigating (multiverse debugging).    |
| `workload`   | `references/workload/MODULE.md`   | Building or iterating on a workload.                                       |
| `properties` | `references/properties/MODULE.md` | Importing existing properties (`import`) or defining a new one (`define`). |

If there is no argument, infer the module from the request. If it is still ambiguous, ask.

## Platform notes

If you are running on one of these platforms, read its notes before starting:

- Devin: `references/platforms/devin.md`
- Replit: `references/platforms/replit.md`
