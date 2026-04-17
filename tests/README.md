# Azure Pipelines Tests

Este directorio contiene tests de comportamiento para plantillas de condicionamiento de jobs:

- `conditioned-job.yml`
- `conditioned-job-wrapper.yml`

Pipeline de pruebas principal:

- [`azure-pipelines-test.yml`](C:\source\repos\github\qckdev-suite\azure-devops-pipelines\tests\azure-pipelines-test.yml)

Plantilla auxiliar usada por el wrapper:

- [`templates/conditioned-wrapper-target-job.yml`](C:\source\repos\github\qckdev-suite\azure-devops-pipelines\tests\templates\conditioned-wrapper-target-job.yml)

## Qué se testea

### Stage `Preparation`

Job `Prepare` / Task `SetOutputs` publica variables para pruebas:

- Outputs:
  - `isTested=true`
  - `isBroken=false`
  - `outBuildId`
  - `outDefinitionName`
  - `outSourceBranch`
- No output:
  - `plainLocalOnly`

## Matriz de tests (Stage `TestSingle`)

| Test | Template | Condición | Objetivo | Resultado esperado |
|---|---|---|---|---|
| `ConditionedJobCheck` | `conditioned-job.yml` | `isTested=true` | Validar ejecución + mapeo de outputs/no-outputs | `Succeeded` |
| `ConditionedJobWrapperCheck` | `conditioned-job-wrapper.yml` | `isTested=true` | Validar paridad funcional del wrapper | `Succeeded` |
| `ConditionedJobBrokenCheck` | `conditioned-job.yml` | `isBroken=false` | Validar que no entra cuando la condicional es false | `Skipped` |
| `ConditionedJobWrapperBrokenCheck` | `conditioned-job-wrapper.yml` | `isBroken=false` | Misma prueba negativa vía wrapper | `Skipped` |
| `VerifyConditionedExecution` | Job nativo | `succeededOrFailed()` | Asertar estados finales esperados de los 4 tests anteriores | `Succeeded` |

## Matriz de tests (Stage `TestAlone`)

Este bloque prueba el mismo comportamiento, pero consumiendo outputs de un job del mismo stage (flujo `dependencies`).

| Test | Template | Condición | Objetivo | Resultado esperado |
|---|---|---|---|---|
| `ConditionedJobCheckAlone` | `conditioned-job.yml` | `isTested=true` | Validar ejecución + mapeo de outputs/no-outputs en Test Alone | `Succeeded` |
| `ConditionedJobWrapperCheckAlone` | `conditioned-job-wrapper.yml` | `isTested=true` | Validar paridad funcional del wrapper en Test Alone | `Succeeded` |
| `ConditionedJobBrokenCheckAlone` | `conditioned-job.yml` | `isBroken=false` | Validar que no entra cuando la condicional es false en Test Alone | `Skipped` |
| `ConditionedJobWrapperBrokenCheckAlone` | `conditioned-job-wrapper.yml` | `isBroken=false` | Misma prueba negativa vía wrapper en Test Alone | `Skipped` |
| `VerifyConditionedExecutionAlone` | Job nativo | `succeededOrFailed()` | Asertar estados finales esperados de los 4 tests de Test Alone | `Succeeded` |

## Validaciones clave

- Variables con `isOutput=true` se pueden mapear y consumir entre stages/jobs.
- Variables sin `isOutput=true` **no** deben aparecer en `mappingVariables`.
- La ejecución condicional por variable (`isTested` / `isBroken`) funciona igual en:
  - `conditioned-job.yml`
  - `conditioned-job-wrapper.yml`
- El stage de test se ejecuta con `succeededOrFailed()` para capturar regresiones aunque existan fallos previos.

## Criterio de fallo

El pipeline se considera incorrecto si ocurre cualquiera de estos casos:

- `ConditionedJobCheck` o `ConditionedJobWrapperCheck` no terminan en `Succeeded`.
- `ConditionedJobBrokenCheck` o `ConditionedJobWrapperBrokenCheck` no terminan en `Skipped`.
- Algún job de validación detecta discrepancias y reporta error.
