# Stage, Job and Task Naming Conventions

## Objetivo
Este documento resume las normas acordadas para definir `stages`, `jobs` y `tasks` en nuestras plantillas de Azure Pipelines, evitando ambiguedades, colisiones de nombres y errores de resolucion entre repositorios.

## 1) Regla general de nombres auto-generados

### Prioridad
1. Si se pasa `stageName`/`jobName`/`taskName` explicito, se usa ese valor tal cual.
2. Si no se pasa, se calcula automaticamente con `default + prefix + suffix`.

### Formato
- Base por defecto (ejemplos): `Preparation`, `BuildAndPush`, `Deploy`, `Prepare`, `DeployToACA`, `ResolveBuildId`, `DetectPublishableChanges`, `DetectDeployableBuilds`.
- Composicion: `Base + PrefixSanitizado + '_' + SuffixSanitizado`.
- Si `prefix` y `suffix` estan vacios, el nombre final es solo `Base`.
- Si falta uno de los dos, no se dejan separadores huerfanos.

### Sanitizacion
Para `stageName`/`jobName`/`taskName` tecnicos:
- Reemplazar `/`, `-`, espacios, `.` y `:` por `_`.
- Mantener solo caracteres validos para Azure (`[A-Za-z0-9_]`).

## 2) Semantica de `prefix/suffix` vs `tagPrefix/tagSuffix`

### Nombres tecnicos
- En `stages` y `jobs` usar solo: `prefix`, `suffix`.

### Tags funcionales
- En `tasks/scripts` que construyen tags usar solo: `tagPrefix`, `tagSuffix`.

### Regla de no-duplicidad
- No mezclar `prefix/suffix` y `tagPrefix/tagSuffix` en el mismo punto de consumo.
- Los wrappers deben resolver antes y pasar al consumidor final solo el par que le corresponde.

## 3) Orden obligatorio de parametros

Cuando estos parametros existan, deben declararse en este orden:

### Inicio (si existen)
1. `jobName`
2. `displayName`

### Final (si existen)
1. `dependsOn`
2. `variables`
3. `additionalVariables`
4. `condition`
5. `additionalCondition`

La misma disciplina aplica en las llamadas a templates.

## 4) Reglas de dependencias y outputs

- Si un `job` o `stage` va a ser referenciado en `dependsOn`, `dependencies.*` o `stageDependencies.*`, conviene fijar nombre explicito para evitar roturas por renombrado automatico.
- Si se usa nombre auto-generado y luego se referencia, la referencia debe usar exactamente el nombre resuelto.
- Si se pasa `jobName` explicito, no debe alterarse concatenando `prefix/suffix`.

## 5) Reglas de repositorio de templates

- Las referencias `template: ...@alias` deben usar alias literal y estable.
- No usar alias dinamico (`@${{ ... }}`) en templates de bajo nivel.
- No usar parametros tipo `innerRepository` para resolver alias en runtime/compile-time.
- Definir y respetar un alias canonico de repositorio en los pipelines consumidores.

## 6) Convencion para producto mono vs multi-componente

### Monocomponente
- Dejar `prefix` y `suffix` vacios cuando no aporten diferenciacion.
- Resultado: nombres estables por defecto (sin sufijos tecnicos innecesarios).

### Multicomponente
- Usar `prefix` para componente (por ejemplo `api`, `ui`).
- Usar `suffix` para entorno o variante (por ejemplo `test`, `pre`, `pro`, `public`).
- Esto evita colisiones cuando conviven varios jobs/stages homonimos.

## 7) Checklist rapido antes de publicar cambios

- No hay `sufix`/`tagSufix` legacy.
- `prefix` aparece antes que `suffix` en declaraciones y llamadas.
- `jobName`/`taskName` explicitos no se recomponen.
- No hay alias dinamicos de repositorio en templates (`@${{ ... }}`).
- Las referencias `dependsOn`/`dependencies`/`stageDependencies` coinciden con nombres reales.
- Si hay dos despliegues similares en un mismo stage (por ejemplo interno/publico), al menos uno tiene `jobName` explicito para evitar duplicados.
