# Properties

Add properties to a workload built with `references/workload/MODULE.md`. If there is none, build it first.

- `define`: add one property the user describes. Follow "Define a property".
- `import`: add properties from a source the user points to. Follow "Import properties".

## Define a property

Input: a property the user wants tested, in their words. For example: "the system recovers within 500 milliseconds once faults stop."

### 1. Make it checkable

Restate the property as one condition you can check in code, and confirm from the system's code and docs that the system promises it. Ask the user about anything the code can't answer, such as what "recovered" means or when the 500 milliseconds start.

Pick the assertion type with "Choosing assertions" in `references/workload/driver.md`.

### 2. Write it down

Write `antithesis/properties/<slug>.md`:

```markdown
# <assertion message>

- **Claim:** <the user's words>
- **Condition:** <what is checked, including any answers the user gave>
- **Checked in:** <where, from step 3>
- **Assertion:** <type>
```

### 3. Add the check

Put the check where it can see the condition:

- **A driver rule**, for what clients see. See `references/workload/driver.md`.
- **The log thread**, for a node's internal state. See `references/workload/system-logs.md`.
- **A quiet period**, for what must hold once faults stop. See "Quiet periods" in `references/workload/driver.md`.
- **The system's code**, only when nothing outside it can see the condition. Tell the user, since it changes their code.

Also add a sometimes or reachable assertion for the situation the property is about, so a pass means the situation happened.

### 4. Check it locally

Run the local loop in `references/workload/local-loop.md`. The property should pass. Then break the condition on purpose, for example with a deadline of zero or a small change to the system, and confirm the property fails. Undo the change.

### 5. Check it in a short run

Launch a 5-minute run as `references/workload/run-loop.md` describes, and read it with the triage module. One run can check several new properties. The property should pass, and its sometimes or reachable assertion should be hit. If it fails, root-cause it with the triage module's `root-cause` mode.

## Import properties

Input: a source of properties, such as a property catalog for this kind of system, the user's documents (a spec, design doc or SLA), existing tests or checks, or past bugs.

Work in batches of up to 10 properties, unless the request says otherwise.

1. List the properties the source states or implies that apply to this system. For a past bug, confirm its cause from the fix, not only the report.
2. Show the user the most important batch, and ask which to add, unless the request already says which.
3. Define each chosen property with steps 1 to 4 of "Define a property", then check them all in one short run (step 5).
4. If the source has more, ask the user whether to import the next batch, unless the request already says.
