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



---

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


# That didn't work. I still variable value as null in teh child pipeline






---

Thanks — that helps. The **actual root cause** here is two things that commonly trip people up in Harness chained pipelines:

1. **The child pipeline variable must be declared as a *runtime input* (value = `<+input>`)** so it appears in the parent pipeline's *Inputs* and can be mapped.
   (If you leave a concrete default like `'not-passed'`, it will not show up for the parent to map as a runtime input.) ([Harness][1])

2. **When reading that value inside the child pipeline at runtime you must reference the pipeline variable** (e.g. `<+pipeline.variables.parentSequenceId>`), not `<+input.*>`. The `<+input>` token is used at **definition time** to mark a runtime input — it’s not the runtime reference syntax. Use Harness expressions to read pipeline variables in steps. ([Harness][2])

---

### Exactly what to change (copy / paste)

**Child pipeline (Regression Test)** — make the pipeline variable a runtime input and reference it via `pipeline.variables` in the S3Upload `target`:

```yaml
pipeline:
  name: Regression Test
  identifier: Regression_Test
  orgIdentifier: ATD
  projectIdentifier: Backstage
  delegateSelectors:
    - che-prod-tools-harness-atd-delegate
  properties:
    ci:
      codebase:
        connectorRef: org.QISD_GHE
        repoName: backstage-regression-tests
        build: <+input>
  variables:
    - name: parentSequenceId
      type: String
      description: Sequence ID of the parent pipeline
      required: true
      value: <+input>    # <<<<--- MUST be runtime input so parent can map it
  stages:
    - stage:
        name: Regression Testing
        identifier: Regression_Testing
        type: CI
        spec:
          cloneCodebase: true
          infrastructure:
            type: KubernetesDirect
            spec:
              connectorRef: account.acct_toolsprod_cluster_connector
              namespace: harness-delegate-ng
              serviceAccountName: harness-atd-delegate
              automountServiceAccountToken: true
              nodeSelector: {}
              harnessImageConnectorRef: account.acct_ecr_eid_connector_prod
              os: Linux
          execution:
            steps:
              - step:
                  type: S3Upload
                  name: Upload Regression test results to ERDS
                  identifier: Upload_Regression_test_results_to_ERDS
                  spec:
                    connectorRef: org.QISD_S3_Storage
                    region: us-east-1
                    bucket: harness-atd-storage
                    sourcePath: playwright-report/index.html
                    # Use pipeline.variables to reference the passed-in value at runtime
                    target: files/<+project.name>/<+pipeline.name>/<+pipeline.variables.parentSequenceId>
```

**Parent pipeline (CI Pipeline)** — map the parent `pipeline.sequenceId` into the child pipeline `inputs.variables`:

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
        identifier: Regression_Test
        variables:
          - name: parentSequenceId
            type: String
            value: <+pipeline.sequenceId>   # passes the parent sequence id into the child
        properties:
          ci:
            codebase:
              build:
                type: branch
                spec:
                  branch: main
```

This pattern matches the Harness docs for chaining pipelines and mapping inputs. ([Harness][1])

---

### Quick verification steps (handy for debugging)

1. **Save both pipelines** after making the changes (child must be saved with `value: <+input>`).
2. In the **parent pipeline UI** → open the chained child stage → check the **Inputs** tab — you should see `parentSequenceId` as a runtime input (expression) and you should be able to set it to an expression there (`<+pipeline.sequenceId>`). ([Harness][1])
3. Add a tiny debug step at the top of the child pipeline to print the variable so you can see resolution in logs:

```yaml
- step:
    type: ShellScript
    name: Debug print parentSequence
    identifier: Debug_Print
    spec:
      shell: Bash
      onDelegate: true
      source:
        type: Inline
        spec:
          script: |
            echo "parentSequenceId (pipeline.variables): <+pipeline.variables.parentSequenceId>"
            echo "parentSequenceId (inputSet): '<+inputSet>'"
```

4. Run the parent pipeline and check the child pipeline execution logs — you should see the printed sequence id.

---

### Common gotchas

* **If you keep a literal default** (e.g. `'not-passed'`) instead of `<+input>` the parent stage won’t render the runtime input for mapping. ([Harness][1])
* **Don’t use `<+input.parentSequenceId>` at runtime** — it’s not the correct expression to *read* the variable inside a step. Use `<+pipeline.variables.parentSequenceId>`. ([Harness][3])
* Make sure the parent has permission to execute the child pipeline and both pipelines are saved in a place Harness can fetch (same account/org/project or accessible across them). ([Harness][1])

---

If you want, I can:

* produce a full, merged YAML file for both pipelines (ready to paste), or
* give you the exact debug step and where to insert it in the child pipeline.

Which would help most right now?

[1]: https://developer.harness.io/docs/platform/pipelines/pipeline-chaining/ "Pipeline chaining in Harness | Harness Developer Hub"
[2]: https://developer.harness.io/docs/platform/variables-and-expressions/runtime-input-usage/?utm_source=chatgpt.com "Use runtime input - Harness Developer Hub"
[3]: https://developer.harness.io/docs/continuous-delivery/x-platform-cd-features/cd-steps/utilities/shell-script-step/?utm_source=chatgpt.com "Shell Script step | Harness Developer Hub"
