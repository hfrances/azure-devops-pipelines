# Component Catalog Current State

## Objetivo
Aclarar el estado actual de la PoC de `dummy-test` y su relacion con la documentacion de arquitectura para que no haya ambiguedades.

## Estado actual (implementado en codigo)
La implementacion activa en `dummy-test` **no usa manifests**.

Flujo actual:
- `Preparation`:
  - Ejecuta deteccion por componente con `detect-publishable-changes-git.yml`.
  - Exporta solo flags globales de stage (`hasAnyBuildCandidate`, `hasAnyDeployCandidate`).
- `BuildAndPush`:
  - Recalcula deteccion por componente con la misma plantilla canonica.
  - Ejecuta build/push segun `ShouldBuild` / `ShouldPublish`.
  - Marca tags canonicos (`compiled` / `published`).
  - Exporta `ComponentShouldDeploy` como output por job para consumo en Deploy.
- `Deploy`:
  - El stage se habilita solo si `Publish != false` y `Deployable == true`.
  - Cada job se condiciona por outputs del stage Build (`SetOutputs.ComponentShouldDeploy`), sin lectura de manifiestos.

Cambios recientes de contrato:
- El parametro runtime principal pasa a llamarse `Publish` (antes `Deploy`) en la PoC con catalogo de componentes.
- Se anade parametro runtime `Deployable` (`displayName: Deploy (if possible)`, default `true`) para habilitar/deshabilitar toda la fase Deploy.
- En niveles de `stage`/`job`, el contrato de `components[]` deja de usar `prefix`/`suffix` para identidad.
- La identidad funcional del componente usa fallback: `name -> component -> ''`.
- Solo para nombres internos de variables/output (tokens), si el nombre resuelto queda vacio, se aplica fallback tecnico: `default`.
- `BuildAndPush` no entra si `Preparation` exporta `hasAnyBuildCandidate=false`.

## Diseño alternativo (documentado, no activo)
La estrategia con manifests sigue documentada para poder reintroducirla en el futuro:
- `manifest.prepare.json` (base)
- `manifest*.build.json` (incremental por componente)

Referencia:
- `component-catalog-poc.md`

## Por que existen ambos enfoques
- El enfoque actual reduce complejidad y ruido de artifacts en la PoC.
- El enfoque con manifests se mantiene documentado porque puede volver a ser util para trazabilidad avanzada o escenarios de orquestacion mas complejos.

## Regla operativa para el equipo
- Si se quiere simplicidad y velocidad: usar el estado actual (sin manifests).
- Si se necesita trazabilidad por fase/componente: reactivar el diseño con manifests segun la documentacion PoC.
