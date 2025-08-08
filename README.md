# FAQs

## Passing variable form parent to child

I see exactly why your `parentSequenceId` ends up as `null` in the child pipeline — in Harness, when you call a **Pipeline** stage to run a child pipeline, the variables from the parent are **not** magically inherited.

You *can* pass them, but the syntax has to match **how Harness expects child pipeline inputs to be mapped**, and your current YAML is close but not quite correct.

---

## Why it’s failing

In your **parent pipeline**:

```yaml
inputs:
  identifier: Regression_Test
  parentSequenceId: <+pipeline.sequenceId>
  properties:
    ci:
      codebase:
        build:
          spec:
            branch: main
          type: branch
```

This `parentSequenceId` line is **not** actually passing the variable into the child unless the child declares it inside `inputSet` / `variables` mapping in the `Pipeline` stage spec.

In your **child pipeline**, you’ve declared:

```yaml
variables:
  - name: parentSequenceId
    type: String
    required: true
```

But Harness will only bind this variable if you explicitly pass it under `inputs` in the **parent’s Pipeline stage** using the `inputSet` format.

---

## Correct way to pass `parentSequenceId`

Here’s what your **parent pipeline**’s `Pipeline` stage should look like:

```yaml
- stage:
    name: Integration Test
    identifier: Integration_Test
    type: Pipeline
    spec:
      org: ATD
      project: Backstage
      pipeline: Regression_Test
      inputs:
        variables:
          - name: parentSequenceId
            type: String
            value: <+pipeline.sequenceId>
        properties:
          ci:
            codebase:
              build:
                type: branch
                spec:
                  branch: main
```

---

### Key points:

1. **`inputs.variables`** → must exactly match the `variables` block in the child pipeline.
2. **Type** → must match (`String` in both).
3. **Value** → `value: <+pipeline.sequenceId>` will inject the parent’s run sequence ID.
4. You can still pass `properties.ci.codebase` for the branch as you are now.

---

## What happens after fixing

When the parent pipeline runs:

* Harness will evaluate `<+pipeline.sequenceId>` in the parent (e.g., `10234`).
* Pass it to the child pipeline’s `parentSequenceId` variable.
* In the child pipeline, `<+input.parentSequenceId>` (or `<+pipeline.variables.parentSequenceId>`) will now have the correct value.

---

If you want, I can rewrite **both** your parent and child pipeline YAML with the corrected mapping so you can just paste it into Harness and test.
Would you like me to do that?


