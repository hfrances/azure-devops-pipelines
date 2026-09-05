# Conditioned Job and Stage Wrappers

These templates turn dependency outputs into reusable execution gates. They
are useful when a pipeline must decide at runtime whether a job or an entire
stage should run, while still passing the values produced by the decision job
to the work that follows.

Templates:

- `scripts-templates/jobs/conditioned-job.yml`
- `scripts-templates/jobs/conditioned-job-wrapper.yml`
- `scripts-templates/stages/conditioned-stage.yml`

## Why Use Them

Without these templates, every consumer has to repeat the same Azure Pipelines
plumbing:

1. declare a dependency on the producer job or stage;
2. choose between `dependencies` and `stageDependencies`;
3. map output variables into local variables;
4. compose the output check with the normal `condition`; and
5. inject the real build, test, or deploy steps.

The conditioned templates centralize that plumbing. A consumer describes the
business decision (`conditionalVariable`), the values it needs
(`mappingVariables`), and its own configuration (`additionalVariables`). The
template generates the dependency, variables, and condition consistently.

They are especially valuable when:

- several jobs consume the same decision made by a prepare/resolve job;
- a monorepo should skip unaffected components without duplicating expressions;
- a promotion or deploy stage depends on a summary produced by an earlier stage;
- the same condition-and-mapping pattern is used by many pipelines; or
- an existing job template must gain dependency-based gating without changing
  that job template's implementation.

For a single task with a simple, local condition, a regular Azure Pipelines
`condition` is usually clearer. These templates pay off when the dependency
and output-variable wiring is part of the pattern.

## Which Template Should I Choose?

| Need | Template | What it provides |
| --- | --- | --- |
| Define the steps directly in the consumer job | `conditioned-job.yml` | A generated job with injected `steps` |
| Reuse an existing job template | `conditioned-job-wrapper.yml` | The same dependency/condition wiring around another job template |
| Gate several jobs as one unit | `conditioned-stage.yml` | A generated stage with injected `jobs` |

Use `conditioned-job.yml` when the work is naturally expressed at the call
site. Use `conditioned-job-wrapper.yml` when the organization already has a
standard job template and the gate should be added without copying or editing
it. Use `conditioned-stage.yml` when the decision should prevent the whole
stage from starting.

## The Core Concepts

### `conditionalVariable`: the execution gate

This is the name of a boolean output published by the producer. When it is
provided, the generated job or stage runs only when that output is `true` and
the base `condition` also succeeds.

The producer must publish the value as an output from a named step:

```yaml
- pwsh: |
    Write-Host "##vso[task.setvariable variable=shouldBuild;isOutput=true]true"
  name: SetOutputs
```

The consumer then references the producer job, step, and output:

```yaml
conditionalJob: Resolve
conditionalTask: SetOutputs
conditionalVariable: shouldBuild
```

When `conditionalVariable` is omitted, the template still handles dependency
and variable mapping, but it does not add an output-based gate. This is useful
when a consumer must always run but needs values calculated by another job.

### `mappingVariables`: dependency outputs as local variables

`mappingVariables` maps the variable names used by the consumer to the output
names published by the producer:

```yaml
mappingVariables:
  ComponentPath: componentPath
  ImageTag: imageTag
```

The left side (`ComponentPath`, `ImageTag`) is the local name available to the
consumer as `$(ComponentPath)` and `$(ImageTag)`. The right side is the output
name from the producer. If both names are the same, the value can be omitted
or set to an empty value.

This keeps consumers independent from the exact expression used to access the
output. The template selects the correct form automatically:

- same-stage job: `dependencies.<job>.outputs[...]`;
- cross-stage job: `stageDependencies.<stage>.<job>.outputs[...]`; and
- conditioned stage: the corresponding stage dependency output path.

The producer output must use `isOutput=true`, and the producer step must have
the name supplied in `conditionalTask`.

### `additionalVariables`: static consumer configuration

Use `additionalVariables` for values that are not dependency outputs, such as
component paths, feature flags, or fixed mode names:

```yaml
additionalVariables:
  ComponentPath: src/orders
  PublishMode: container
```

These variables are injected directly into the generated job or stage. They
are deliberately separate from `mappingVariables`: the former contains static
configuration, while the latter contains runtime values from another job or
stage.

For `conditioned-stage.yml`, `additionalVariables` can also contain normal
stage variable entries, including variable groups:

```yaml
additionalVariables:
  - group: shared-deploy-settings
  - name: EnvironmentName
    value: test
```

### `conditionalStage`, `conditionalJob`, and `conditionalTask`: the source

Together these identify where the gate and mapped outputs come from:

- `conditionalStage` is optional for `conditioned-job.yml`; empty means the
  producer job is in the same stage. It is required in practice when a job
  consumes outputs from a previous stage.
- `conditionalJob` is the producer job.
- `conditionalTask` is the named step that publishes the outputs. Its default
  is `SetOutputs` in the direct job and stage templates.

For a conditioned stage, `conditionalStage` identifies the previous stage. A
conditioned stage automatically depends on that stage. A conditioned job that
uses a cross-stage source must be placed in a stage that already depends on the
source stage; jobs cannot depend directly on a job in another stage.

### `dependsOn`: extra synchronization

The templates add the source dependency automatically when the source is in
the same stage, and `conditioned-stage.yml` adds the source stage
automatically. Use `dependsOn` for other jobs or stages that must complete
before the consumer starts:

```yaml
dependsOn:
  - SecurityScan
  - PublishMetadata
```

This is for synchronization, not for output mapping. Put output names in
`mappingVariables`.

### `condition`: the base guard

`condition` remains the normal Azure Pipelines expression and defaults to
`succeeded()`. When `conditionalVariable` is configured, the template combines
both rules with `and(...)`:

```text
and(<condition>, eq(<dependency output>, 'true'))
```

For example, `condition: always()` still allows the consumer to be considered
after an earlier failure, but the `conditionalVariable` must still be `true`.

## Example: Gate a Job in the Same Stage

The prepare job publishes a decision and data. The conditioned job consumes
both without writing a `dependencies` expression:

```yaml
stages:
- stage: Build
  jobs:
  - job: Resolve
    steps:
    - pwsh: |
        Write-Host "##vso[task.setvariable variable=shouldBuild;isOutput=true]true"
        Write-Host "##vso[task.setvariable variable=componentPath;isOutput=true]src/orders"
        Write-Host "##vso[task.setvariable variable=imageTag;isOutput=true]orders-123"
      name: SetOutputs

  - template: /scripts-templates/jobs/conditioned-job.yml
    parameters:
      jobName: BuildOrders
      displayName: Build orders component
      conditionalJob: Resolve
      conditionalTask: SetOutputs
      conditionalVariable: shouldBuild
      mappingVariables:
        ComponentPath: componentPath
        ImageTag: imageTag
      additionalVariables:
        BuildConfiguration: Release
      steps:
      - pwsh: |
          Write-Host "Building $(ComponentPath) with tag $(ImageTag)"
          Write-Host "Configuration: $(BuildConfiguration)"
```

The generated `BuildOrders` job automatically depends on `Resolve`. If
`shouldBuild` is `false`, the job is skipped and no build agent work is
performed.

## Example: Gate a Complete Stage

Use a conditioned stage when several jobs share the same gate or when no job in
the stage should start unless the previous stage permits it:

```yaml
- template: /scripts-templates/stages/conditioned-stage.yml
  parameters:
    stageName: DeployOrders
    displayName: Deploy orders
    conditionalStage: Build
    conditionalJob: Resolve
    conditionalTask: SetOutputs
    conditionalVariable: shouldDeploy
    mappingVariables:
      ImageTag: imageTag
    additionalVariables:
      - group: shared-deploy-settings
      - name: EnvironmentName
        value: test
    dependsOn:
      - SecurityScan
    jobs:
      - job: Deploy
        steps:
          - pwsh: |
              Write-Host "Deploying $(ImageTag) to $(EnvironmentName)"
```

The stage depends on `Build` and `SecurityScan`. It runs only when
`Build.Resolve.SetOutputs.shouldDeploy` is `true` and the configured base
condition succeeds. All jobs inside the stage inherit the stage-level decision.

## Example: Add Gating Around an Existing Job Template

Use the wrapper when the actual job already lives in a reusable template:

```yaml
- template: /scripts-templates/jobs/conditioned-job-wrapper.yml
  parameters:
    template: /jobs-templates/build-component-job.yml
    conditionalStage: Prepare
    conditionalJob: Resolve
    conditionalTask: SetOutputs
    conditionalVariable: shouldBuild
    mappingVariables:
      ComponentPath: componentPath
      ImageTag: imageTag
    additionalVariables:
      BuildConfiguration: Release
    parameters:
      componentName: orders
      publishArtifacts: true
```

The wrapped job template receives the generated `variables` and `condition`
alongside its normal parameters. The target job template must expose those
parameters (or otherwise be designed to accept them). This pattern lets the
standard build implementation remain unchanged while every caller gets the
same dependency behavior.

## Parameter Order and Structural Inputs

The templates use a consistent parameter layout:

1. job/stage identifier and `displayName`;
2. source dependency and output mapping parameters;
3. `dependsOn`;
4. `condition`; and
5. structural `steps` (`stepList`) or `jobs` (`jobList`).

Keeping structural collections after `condition` makes the templates easier to
scan and keeps the execution policy separate from the work being injected.

## Common Mistakes

- Publishing the value without `isOutput=true`.
- Omitting the `name` from the producer step or using a different value in
  `conditionalTask`.
- Putting a runtime dependency output in `additionalVariables` instead of
  `mappingVariables`.
- Expecting a cross-stage conditioned job to create a stage dependency by
  itself; the containing stage must depend on the source stage.
- Using a conditioned stage when only one simple task needs a condition.
- Publishing `True`/`False` inconsistently when consumers compare the output
  with the string `'true'`.

The test pipeline covers positive and negative job conditions, wrapper parity,
variable mapping, additional variables, and stage conditions.
