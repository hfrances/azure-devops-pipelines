# Stage, Job and Task Naming Conventions

## Tabla de contenido
- [Objetivo](#objetivo)
- [1) Regla general de nombres auto-generados](#1-regla-general-de-nombres-auto-generados)
- [2) Semantica de `prefix/suffix` vs `tagPrefix/tagSuffix`](#2-semantica-de-prefixsuffix-vs-tagprefixtagsuffix)
- [3) Orden obligatorio de parametros](#3-orden-obligatorio-de-parametros)
- [4) Reglas de dependencias y outputs](#4-reglas-de-dependencias-y-outputs)
- [5) Reglas de repositorio de templates](#5-reglas-de-repositorio-de-templates)
- [6) Orden de variables en pipeline (cuando existan)](#6-orden-de-variables-en-pipeline-cuando-existan)
- [7) Convencion para producto mono vs multi-componente](#7-convencion-para-producto-mono-vs-multi-componente)
- [8) Checklist rapido antes de publicar cambios](#8-checklist-rapido-antes-de-publicar-cambios)

## Objetivo
Este documento resume las normas acordadas para definir `stages`, `jobs` y `tasks` en plantillas de Azure Pipelines, evitando ambiguedades, colisiones de nombres y errores de resolucion entre repositorios.

## 1) Regla general de nombres auto-generados

### Prioridad
- Si se pasa `stageName`/`jobName`/`taskName` explicito, se usa ese valor tal cual.
- Si no se pasa, se calcula automaticamente con `default + prefix + suffix`.

### Formato
- Base por defecto (ejemplos): `Preparation`, `BuildAndPush`, `Deploy`, `Prepare`, `DeployToACA`, `ResolveBuildId`, `DetectPublishableChanges`, `DetectDeployableBuilds`.
- Composicion: `Base + PrefixSanitizado + '_' + SuffixSanitizado`.
- Si `prefix` y `suffix` estan vacios, el nombre final es solo `Base`.
- Si falta uno de los dos, no se dejan separadores huerfanos.

### Sanitizacion
Para `stageName`/`jobName`/`taskName` tecnicos:
- Reemplazar `/`, `-`, espacios, `.` y `:` por `_`.
- Mantener solo caracteres validos para Azure (`[A-Za-z0-9_]`).

## 2) Semantica de identidad de componente vs `tagPrefix/tagSuffix`

### Nombres tecnicos de stage/job
- En catalogos `components[]`, la identidad funcional del componente debe salir de:
  - `name` (preferente)
  - `component` (fallback funcional)
  - `''` para valores visibles cuando ambos faltan.
- Si hace falta un token tecnico no vacio para outputs/nombres internos, usar fallback tecnico `default`.

### Tags funcionales
- En `tasks/scripts` que construyen tags usar `tagPrefix`, `tagSuffix` (o `prefix/suffix` si la task legacy lo exige).

### Regla de no-duplicidad
- No mezclar en el mismo punto de consumo una identidad funcional (`name/component`) con un prefijo tecnico de tag.
- Los wrappers deben resolver antes y pasar al consumidor final solo el dato que le corresponde.

## 3) Orden obligatorio de parametros

Cuando estos parametros existan, deben declararse en este orden.

### Inicio (si existen)
- `jobName`
- `displayName`

### Final (si existen)
- `dependsOn`
- `variables`
- `additionalVariables`
- `condition`
- `additionalCondition`

La misma disciplina aplica en las llamadas a templates.

## 4) Reglas de dependencias y outputs

- Si un `job` o `stage` va a ser referenciado en `dependsOn`, `dependencies.*` o `stageDependencies.*`, conviene fijar nombre explicito para evitar roturas por renombrado automatico.
- Si se usa nombre auto-generado y luego se referencia, la referencia debe usar exactamente el nombre resuelto.
- Si se pasa `jobName` explicito, no debe alterarse concatenando `prefix/suffix`.
- Si un output se exporta desde un step con `name: <StepName>`, la lectura debe usar exactamente `outputs['<StepName>.<Variable>']` (ejemplo: `SetOutputs.ComponentShouldDeploy`).

## 5) Reglas de repositorio de templates

- Las referencias `template: ...@alias` deben usar alias literal y estable.
- No usar alias dinamico (`@${{ ... }}`) en templates de bajo nivel.
- No usar parametros tipo `innerRepository` para resolver alias en runtime/compile-time.
- Definir y respetar un alias canonico de repositorio en los pipelines consumidores.

## 6) Orden de variables en pipeline (cuando existan)

En bloques `variables:` de pipelines consumidores, mantener este orden exacto.

### Grupo 1: Variables globales de ejecucion
- `dockerRegistryServiceConnection`
- `azureServiceConnection`
- `identityName`

### Grupo 2: Infraestructura comun
- `acrName`
- `resourceGroup`
- `resourceGroupVNet`
- `vnetName`
- `subnetName`
- `caeLocation`

### Grupo 3: Repositorios de imagen
- Mono-componente: `imageRepository`.
- Multi-componente: `<component1>ImageRepository`, `<component2>ImageRepository`, `<component3>ImageRepository`.

Ejemplos:
- combi: `backImageRepository`, `frontImageRepository`, `frontNextImageRepository`
- simple: `app1ImageRepository`, `app2ImageRepository`

### Grupo 4: Nombres compartidos de CAE/servicios
- `caeName`
- `liveAcaName`

### Grupo 5: Variables por componente
Para cada componente, declarar en este orden exacto (solo en multi-componente):
- `<component>AcaName`
- `<component>IngressType`
- `<component>TargetPort`
- `<component>Cpu`
- `<component>Memory`
- `<component>MinReplicas`
- `<component>MaxReplicas`

Orden recomendado de componentes en todo el archivo:
- API / back
- UI / front
- UI Next / frontNext

### Regla de prefijo de componente
- Si el pipeline es mono-componente, no usar prefijo de componente (ejemplo: `imageRepository`).
- Si el pipeline es multi-componente, usar prefijo obligatorio en `camelCase` (primera letra en minuscula), por ejemplo: `back`, `front`, `frontNext`, `app1`, `app2`.
- Mantener el mismo prefijo de componente de forma consistente en todas las variables relacionadas.

### Variables unificadas (excepcion de diseno)
Si varias propiedades son identicas para todos los componentes, se permite definirlas una sola vez (sin prefijo), por ejemplo:
- `acaTargetPort`
- `acaCpu`
- `acaMemory`
- `acaMinReplicas`
- `acaMaxReplicas`

En ese caso, no duplicar las mismas propiedades por componente.

### Ejemplos rapidos
- Valido (mono): `imageRepository`, `caeName`, `acaTargetPort`
- Valido (multi con unificadas): `app1AcaName`, `app2AcaName`, `acaTargetPort`
- No valido: mezclar `imageRepository` con `app1ImageRepository`/`app2ImageRepository` en el mismo pipeline multi-componente.

### Notas
- Si un grupo no aplica, se omite completo.
- Si una variable no aplica, se omite sin reordenar las demas.
- Mantener separacion por lineas en blanco entre grupos.

## 7) Convencion para producto mono vs multi-componente

### Monocomponente
- Dejar la identidad funcional vacia (`name/component`) cuando no aporte diferenciacion visible.
- Resultado: nombres estables por defecto (sin sufijos tecnicos innecesarios).

### Multicomponente
- Usar `name` (o `component` como fallback) para identificar cada componente.
- Usar `suffix` solo donde realmente aplique como variante tecnica del job/stage.
- Esto evita colisiones cuando conviven varios jobs/stages homonimos.

## 8) Checklist rapido antes de publicar cambios

- No hay `sufix`/`tagSufix` legacy.
- La identidad funcional (`name/component`) no depende de `prefix`.
- `jobName`/`taskName` explicitos no se recomponen.
- No hay alias dinamicos de repositorio en templates (`@${{ ... }}`).
- Las referencias `dependsOn`/`dependencies`/`stageDependencies` coinciden con nombres reales.
- Si hay dos despliegues similares en un mismo stage (por ejemplo interno/publico), al menos uno tiene `jobName` explicito para evitar duplicados.
