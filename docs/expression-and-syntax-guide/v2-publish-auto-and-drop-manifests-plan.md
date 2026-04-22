# v2.0 Publish auto y manifests en drop

## Objetivo

Restaurar en `v2.0` el comportamiento de `Publish=auto|true|false` para que la publicación dependa de la rama cuando corresponda, con un valor por defecto compartido y sobreescribible por pipeline. Además, mover los `deployment-manifest` al artifact `drop` con nombre `deployment-manifest-<component>.json`.

## Decisiones iniciales

- `Publish=false`: no publica ni despliega.
- `Publish=true`: publica si hay cambios detectados.
- `Publish=auto`: publica si hay cambios detectados y la rama actual está dentro de `publishEnabledBranches`.
- El cálculo efectivo se resolverá en `Preparation` y se propagará como output para no duplicar lógica en cada pipeline consumidor.
- El stage `Deploy` seguirá bloqueándose por `BuildSummary.HasAnyComponentShouldDeploy`, de forma que los consumidores actuales no necesiten cambios inmediatos.
- Los manifests dejarán de publicarse como artifacts separados y pasarán a escribirse en `drop`.

## Implementación por fases

### Fase 1

1. Añadir `publishEnabledBranches` a [hmygroup/azure-pipelines-templates/v2.0/templates/components-pipeline-stages.yml](../../../../../hmygroup/azure-pipelines-templates/v2.0/templates/components-pipeline-stages.yml).
2. Resolver `publishEnabled` en [hmygroup/azure-pipelines-templates/v2.0/templates/prepare-components-job.yml](../../../../../hmygroup/azure-pipelines-templates/v2.0/templates/prepare-components-job.yml).
3. Convertir `ComponentShouldPublish_*` en el valor efectivo final, no sólo el detectado por cambios.
4. Hacer que [hmygroup/azure-pipelines-templates/v2.0/templates/build-components-jobs.yml](../../../../../hmygroup/azure-pipelines-templates/v2.0/templates/build-components-jobs.yml) reutilice ese valor efectivo para `Push image`, `mark-build-published` y `BuildSummary`.
5. Guardar `deployment-manifest-<component>.json` en `$(Build.ArtifactStagingDirectory)/drop`.
6. Eliminar la publicación de artifacts `deployment-manifest-*` separados.

### Fase 2

1. Exponer override por pipeline en los `azure-pipelines.yml` de ejemplo.
2. Actualizar documentación desalineada.
3. Revisar consumidores externos del contrato anterior de manifests.
4. Marcar plantillas vacías, copias o wrappers viejos como obsoletos.

## Riesgos

- Cualquier consumidor que descargue `deployment-manifest-*` como artifact separado tendrá que pasar a descargar `drop`.
- Algunos pipelines consumidores pueden seguir usando condiciones propias de `Deploy`; aunque el bloqueo por `BuildSummary` mantendrá compatibilidad, conviene alinearlos después.

## Estado

- Documento creado.
- Fase 1 iniciada en las plantillas compartidas.
