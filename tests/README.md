# Azure Pipelines Tests

Este directorio contiene tests de comportamiento para plantillas de condicionamiento de jobs:

- `conditioned-job.yml`
- `conditioned-job-wrapper.yml`
- `conditioned-stage.yml`

Pipeline de pruebas principal:

- [`azure-pipelines-test.yml`](C:\source\repos\github\qckdev-suite\azure-devops-pipelines\tests\azure-pipelines-test.yml)

Plantilla auxiliar usada por el wrapper:

- [`templates/conditioned-wrapper-target-job.yml`](C:\source\repos\github\qckdev-suite\azure-devops-pipelines\tests\templates\conditioned-wrapper-target-job.yml)

## Tabla de contenido

- [Matriz de tests (`TestAlone`)](#matriz-de-tests-testalone)
- [Matriz de tests (`TestSingle`)](#matriz-de-tests-testsingle)
- [Matriz de tests (`TestCombi`)](#matriz-de-tests-testcombi)
- [Stages template (`StageSingle` y `StageCombi`)](#stages-template-stagesingle-y-stagecombi)
- [Verificación final de stages](#verificación-final-de-stages)
- [Parámetros de inversión](#parámetros-de-inversión)
- [Validaciones clave](#validaciones-clave)
- [Criterio de fallo](#criterio-de-fallo)

## Matriz de tests (`TestAlone`)

Este bloque prueba el mismo comportamiento, pero consumiendo outputs de un job del mismo stage (flujo `dependencies`).

| Test | Template | Condición | Objetivo | Resultado esperado |
|---|---|---|---|---|
| `ConditionedJobCheckAlone` | `conditioned-job.yml` | `isTested=true` | Validar ejecución + mapeo de outputs/no-outputs en Test Alone | `Succeeded` |
| `ConditionedJobWrapperCheckAlone` | `conditioned-job-wrapper.yml` | `isTested=true` | Validar paridad funcional del wrapper en Test Alone | `Succeeded` |
| `ConditionedJobBrokenCheckAlone` | `conditioned-job.yml` | `isBroken=false` | Validar que no entra cuando la condicional es false en Test Alone | `Skipped` |
| `ConditionedJobWrapperBrokenCheckAlone` | `conditioned-job-wrapper.yml` | `isBroken=false` | Misma prueba negativa vía wrapper en Test Alone | `Skipped` |
| `VerifyConditionedExecutionAlone` | Job nativo | `not(canceled())` | Asertar estados finales esperados de los 4 tests de Test Alone | `Succeeded` |

## Matriz de tests (`TestSingle`)

Preparación incluida dentro del caso (`PreparationSingle`):

- Job `Prepare` / Task `SetOutputs` publica variables para pruebas:
  - Outputs:
    - `isTested=true`
    - `isBroken=false`
    - `outBuildId`
    - `outDefinitionName`
    - `outSourceBranch`
  - No output:
    - `plainLocalOnly`

| Test | Template | Condición | Objetivo | Resultado esperado |
|---|---|---|---|---|
| `ConditionedJobCheck` | `conditioned-job.yml` | `isTested=true` | Validar ejecución + mapeo de outputs/no-outputs | `Succeeded` |
| `ConditionedJobWrapperCheck` | `conditioned-job-wrapper.yml` | `isTested=true` | Validar paridad funcional del wrapper | `Succeeded` |
| `ConditionedJobBrokenCheck` | `conditioned-job.yml` | `isBroken=false` | Validar que no entra cuando la condicional es false | `Skipped` |
| `ConditionedJobWrapperBrokenCheck` | `conditioned-job-wrapper.yml` | `isBroken=false` | Misma prueba negativa vía wrapper | `Skipped` |
| `VerifyConditionedExecution` | Job nativo | `not(canceled())` | Asertar estados finales esperados de los 4 tests del caso | `Succeeded` |

## Matriz de tests (`TestCombi`)

Preparación incluida dentro del caso (`PrepareCombi`):

- `SetOutputsA`: genera outputs para A (`isTestedA`, `isBrokenA`, `out*A`) y un no-output (`plainLocalOnlyA`)
- `SetOutputsB`: genera outputs para B (`isTestedB`, `isBrokenB`, `out*B`) y un no-output (`plainLocalOnlyB`)
- `SetSummary`: calcula agregados para condicionar la entrada al caso:
  - `isTestedSummary = isTestedA OR isTestedB`
  - `isBrokenSummary = isBrokenA AND isBrokenB`

`TestCombi` ejecuta la paridad A/B usando las variables de `PreparationCombi`.

| Test | Template | Condición | Objetivo | Resultado esperado |
|---|---|---|---|---|
| `ConditionedJobCheckA` | `conditioned-job.yml` | `isTestedA=true` | Validar ejecución + mapeo outputs/no-output de A | `Succeeded` |
| `ConditionedJobWrapperCheckA` | `conditioned-job-wrapper.yml` | `isTestedA=true` | Validar paridad wrapper para A | `Succeeded` |
| `ConditionedJobBrokenCheckA` | `conditioned-job.yml` | `isBrokenA=false` | Validar que no entra en negativo de A | `Skipped` |
| `ConditionedJobWrapperBrokenCheckA` | `conditioned-job-wrapper.yml` | `isBrokenA=false` | Negativo de A vía wrapper | `Skipped` |
| `ConditionedJobCheckB` | `conditioned-job.yml` | `isTestedB=true` | Validar ejecución + mapeo outputs/no-output de B | `Succeeded` |
| `ConditionedJobWrapperCheckB` | `conditioned-job-wrapper.yml` | `isTestedB=true` | Validar paridad wrapper para B | `Succeeded` |
| `ConditionedJobBrokenCheckB` | `conditioned-job.yml` | `isBrokenB=false` | Validar que no entra en negativo de B | `Skipped` |
| `ConditionedJobWrapperBrokenCheckB` | `conditioned-job-wrapper.yml` | `isBrokenB=false` | Negativo de B vía wrapper | `Skipped` |
| `VerifyConditionedExecutionCombi` | Job nativo | `not(canceled())` | Asertar estados finales esperados del caso A/B | `Succeeded` |

## Stages template (`StageSingle` y `StageCombi`)

Además de los tests de jobs, el pipeline ejecuta dos stages creados con la template `conditioned-stage.yml`:

- `StageSingle`:
  - Depende de `PreparationSingle`.
  - Condición por output: `PreparationSingle.Prepare.SetOutputs.isTested`.
  - Incluye `StageSingleMarker` y publica output `stageSingleExecuted=true`.
- `StageCombi`:
  - Depende de `PreparationCombi`.
  - Condición por output: `PreparationCombi.PrepareCombi.SetSummary.isTestedSummary`.
  - Incluye `StageCombiMarker` y publica output `stageCombiExecuted=true`.

## Verificación final de stages

Stage `VerifyTemplatedStages` valida explícitamente que:

- `StageSingle` terminó en `Succeeded`.
- `StageCombi` terminó en `Succeeded`.
- Los outputs de marker existen y valen `true`:
  - `stageSingleExecuted`
  - `stageCombiExecuted`

## Parámetros de inversión

El pipeline expone estos parámetros para forzar escenarios:

- `invertConditionFlags`:
  - Aplica a `TestAlone` y `TestSingle`.
  - Invierte `isTested`/`isBroken`.
- `invertConditionFlagsCombiA`:
  - Aplica solo al bloque `A` de `PreparationCombi`.
  - Invierte `isTestedA`/`isBrokenA`.
- `invertConditionFlagsCombiB`:
  - Aplica solo al bloque `B` de `PreparationCombi`.
  - Invierte `isTestedB`/`isBrokenB`.

## Validaciones clave

- Variables con `isOutput=true` se pueden mapear y consumir entre stages/jobs.
- Variables sin `isOutput=true` **no** deben aparecer en `mappingVariables`.
- La ejecución condicional por variable (`isTested` / `isBroken`) funciona igual en:
  - `conditioned-job.yml`
  - `conditioned-job-wrapper.yml`
- La ejecución condicional a nivel stage también se valida con `conditioned-stage.yml`.
- El stage de test se ejecuta con `not(canceled())` para capturar regresiones aunque existan fallos previos.

## Criterio de fallo

El pipeline se considera incorrecto si ocurre cualquiera de estos casos:

- Cualquier check positivo (`Check*`) no termina en `Succeeded`.
- Cualquier check negativo (`BrokenCheck*`) no termina en `Skipped`.
- Algún job de validación detecta discrepancias y reporta error.

