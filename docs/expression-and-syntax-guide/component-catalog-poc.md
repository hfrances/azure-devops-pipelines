# Component Catalog PoC for Azure Pipelines

## Tabla de contenido
- [1) Idea base (nivel inicial)](#1-idea-base-nivel-inicial)
- [2) Problema que resolvemos](#2-problema-que-resolvemos)
- [3) Decision de diseño](#3-decision-de-diseno)
- [4) Regla practica del equipo](#4-regla-practica-del-equipo)
- [5) Estructura aplicada en la PoC](#5-estructura-aplicada-en-la-poc)
- [6) Flujo de ejecucion](#6-flujo-de-ejecucion)
- [7) Manifiestos: nomenclatura y contenido](#7-manifiestos-nomenclatura-y-contenido)
- [8) Ejemplo recomendado (root visible, UX limpia)](#8-ejemplo-recomendado-root-visible-ux-limpia)
- [9) Guia por perfil](#9-guia-por-perfil)
- [10) Casos de uso](#10-casos-de-uso)
- [11) Limites y riesgos](#11-limites-y-riesgos)
- [12) Siguientes pasos](#12-siguientes-pasos)
- [13) Anexo tecnico (informacion ampliada)](#13-anexo-tecnico-informacion-ampliada)
- [14) Ejemplos de consumo por nivel (detalle)](#14-ejemplos-de-consumo-por-nivel-detalle)
- [15) Reglas de resolucion de rutas Docker](#15-reglas-de-resolucion-de-rutas-docker)

## 1) Idea base (nivel inicial)
Queremos declarar los componentes (`app1`, `app2`, `api`, `ui`, etc.) **una sola vez** y usar esa misma definicion para:
- `Preparation`
- `BuildAndPush`
- `Deploy`

Objetivo: evitar copy/paste y mantener una fuente de verdad unica.

## 2) Problema que resolvemos
En Azure DevOps, si declaras `components` en `parameters:` del YAML raiz, aparece en la ventana de **Run pipeline** como parametro editable.

Eso no nos interesa porque:
- ensucia la UX de ejecucion manual,
- permite cambios ad-hoc sobre una estructura que deberia ser estable,
- introduce riesgo operativo.

## 3) Decision de diseño
Usamos este patron:
- En el YAML raiz solo dejamos parametros de control (`Publish`, `Deployable`, `PublishChangeDetection`).
- En el YAML raiz mantenemos el catalogo `components` visible en codigo (para lectura rapida).
- El catalogo se pasa a un template orquestador: `components-pipeline-stages.yml`.
- El template orquestador genera los 3 stages y reparte `components` a los jobs/templates de cada fase.

Resultado:
- `components` se ve en el fichero root (bueno para mantenimiento).
- `components` no aparece editable en UX (bueno para operacion).
- no hay duplicacion del mismo bloque en varios stages (bueno para calidad).

## 4) Regla practica del equipo
Si un pipeline usa catalogo de componentes, recomendamos esta regla:
- **Pipeline root**: control de ejecucion + declaracion visible del catalogo.
- **Template de stages**: orquestacion comun (`Preparation`, `BuildAndPush`, `Deploy`).
- **Templates de job**: logica tecnica por fase.

No es obligatorio para todos los pipelines, pero para pipelines multi-componente es el patron mas limpio.

## 5) Estructura aplicada en la PoC
- `.azuredevops/azure-pipelines-components-poc.yml`
- `.azuredevops/templates/components-pipeline-stages.yml`
- `.azuredevops/templates/prepare-components-job.yml`
- `.azuredevops/templates/build-components-jobs.yml`
- `.azuredevops/templates/deploy-components-jobs.yml`

## 6) Flujo de ejecucion
- `Preparation`: calcula flags por componente y exporta agregados de stage (`hasAnyBuildCandidate`, `hasAnyDeployCandidate`).
- `BuildAndPush`: itera componentes, compila cuando `ShouldBuild=true`, publica cuando `ShouldPublish=true`, y exporta `SetOutputs.ComponentShouldDeploy` por job.
- `Deploy`: el stage entra solo si `Publish != false` y `Deployable == true`; dentro del stage, cada job ejecuta solo cuando el output del job de build correspondiente indica `ComponentShouldDeploy=true`.

Todos consumen el mismo contrato de `components`.


### Deteccion de build/publish en la PoC
La PoC usa la plantilla canonica `detect-publishable-changes-git.yml` por componente.
- Si hay cambios dentro de `workingDirectory` del componente, se marca:
  - `shouldBuild=true`
  - `shouldPublish=true`
  - `shouldDeploy=true`
- Si no hay cambios, esos flags quedan en `false`.

Fallback controlado:
- La plantilla canonica aplica fail-open si la deteccion falla (no bloquea la pipeline).
- El flag `PublishChangeDetection` permite activar/desactivar esta deteccion.
## 7) Manifiestos: nomenclatura y contenido

### Objetivo de los manifiestos
En esta PoC usamos manifiestos para desacoplar fases:
- `Prepare` decide el estado base por componente.
- `BuildAndPush` añade metadatos de build/publicacion por componente.
- `Deploy` consume decisiones sin asumir que todos los componentes se publicaron en el mismo run.

### Nomenclatura

#### 1) Manifest de Prepare (unico)
- Nombre fijo: `manifest.prepare.json`
- Ubicacion: artifact de `prepare`.
- Es unico para el run (no se particiona por componente).

#### 2) Manifest de Build (por componente)
Regla de nombre:
- Si hay `prefix`: `manifest-<prefix>.build.json`
- Si no hay `prefix` y hay mas de un componente: `manifest-<name>.build.json`
- Si no: `manifest.build.json`
- Si hay `suffix`, se anade al final del nombre base:
  - `manifest-<prefix>-<suffix>.build.json`
  - `manifest-<name>-<suffix>.build.json`
  - `manifest-<suffix>.build.json` (caso sin prefijo ni nombre aplicable)

### Contenido

#### `manifest.prepare.json`
Incluye informacion base del run y la lista de componentes:
- `buildId`
- `sourceBranch`
- `sourceVersion`
- `components[]` con campos como:
  - `name`
  - `prefix`
  - `suffix` (si aplica)
  - `workingDirectory`
  - `imageRepository`
  - `dockerfilePath`
  - `dockerBuildContext`
  - `acaName`
  - `shouldBuild`
  - `shouldPublish`
  - `shouldDeploy`
  - `changeDetectionEnabled`

#### `manifest*.build.json` (incremental)
Es incremental respecto al prepare del componente:
- Parte del objeto del componente que viene en `manifest.prepare.json`.
- Anade datos propios de la fase build:
  - `phase: "build"`
  - `imageTag`
  - `buildId`
  - `buildNumber`
  - `sourceVersion`
  - flags finales de `shouldBuild`, `shouldPublish`, `shouldDeploy`

### Regla de seleccion de componente
En escenarios multicomponente, la resolucion del elemento usa:
1. `name` (preferente)
2. `component` (fallback funcional)
3. `''` (vacio) para valores visibles cuando no hay identidad declarada

Esto evita ambiguedades de nomenclatura cuando `name` es la identidad funcional del componente.

En templates de task/job, cuando se consume una propiedad `component` para resolver outputs/tokens, se debe pasar ya como valor resuelto:
- `coalesce(component.name, component.component, '')`

Nota:
- Solo para nombres internos de variables/output (tokens), si la identidad queda vacia se usa fallback tecnico `default`.

### Nota de concurrencia
No se actualiza un unico fichero compartido en paralelo.
Patron aplicado:
- fichero base unico en `Prepare`;
- fichero incremental por componente en `BuildAndPush`.

Asi se evitan condiciones de carrera entre jobs paralelos.



## 8) Ejemplo recomendado (root visible, UX limpia)
```yaml
parameters:
  - name: Publish
    type: string
    values: [auto, true, false]
    default: true
  - name: Deployable
    displayName: Deploy (if possible)
    type: boolean
    default: true
  - name: PublishChangeDetection
    type: boolean
    default: true

stages:
  - template: templates/components-pipeline-stages.yml
    parameters:
      Publish: ${{ parameters.Publish }}
      Deployable: ${{ parameters.Deployable }}
      PublishChangeDetection: ${{ parameters.PublishChangeDetection }}
      components:
        - name: app1
          workingDirectory: src/services/dummy-test-app-1
          imageRepository: dummy/mi-proyecto-test-app-1
          dockerfilePath: $(Build.SourcesDirectory)/src/services/dummy-test-app-1/Dockerfile
          dockerBuildContext: $(Build.SourcesDirectory)/src/services/dummy-test-app-1
          acaName: ca-hmy-dummy-test-app-1
        - name: app2
          workingDirectory: src/services/dummy-test-app-2
          imageRepository: dummy/mi-proyecto-test-app-2
          dockerfilePath: $(Build.SourcesDirectory)/src/services/dummy-test-app-2/Dockerfile
          dockerBuildContext: $(Build.SourcesDirectory)/src/services/dummy-test-app-2
          acaName: ca-hmy-dummy-test-app-2
```

## 9) Guia por perfil
### Para una persona junior
- Piensa en `components` como una lista de apps.
- La escribes una vez.
- El sistema reutiliza esa lista para preparar, compilar y desplegar.
- Si anades `app3`, no copias 3 bloques grandes de YAML: actualizas el catalogo.

### Para un perfil de infra/operacion
- Se reduce superficie de error manual en ejecucion.
- Se mantiene gobernanza: parametros operativos visibles, catalogo tecnico estable.
- Menos drift entre etapas porque todas consumen el mismo objeto.

### Para otro modelo de IA
- Contrato recomendado: `components[]` como objeto canonico compile-time.
- Patrón de expansion: `${{ each component in parameters.components }}`.
- Evitar `components` como runtime parameter del root cuando se quiera UX no editable.
- Separacion de responsabilidades: root (entrada), stage wrapper (orquestacion), job templates (implementacion).

## 10) Casos de uso
- Monocomponente: lista con un unico elemento.
- Multicomponente: `api/ui`, `app1/app2`, etc.
- Escalado: alta de componente por datos, no por duplicacion de estructura.

## 11) Limites y riesgos
- Es una PoC: falta validar completamente todos los casos runtime en pipelines reales.
- Si se abusa del wrapper sin convenciones claras, puede esconder complejidad.
- Conviene documentar contrato minimo obligatorio de `components`.

## 12) Siguientes pasos
- Definir validaciones de contrato (`name`/`component`, `imageRepository`, `acaName`, etc.).
- Introducir flags por componente (`enabled`, `deployEnabled`).
- Elevar el patron a templates compartidas cuando se confirme estable.

## 13) Anexo tecnico (informacion ampliada)

### Patron no soportado directamente por Azure Pipelines
El siguiente formato no es un patron soportado como DSL general dentro de `parameters.components`:

```yaml
components:
  - template: ...
```

Por eso usamos `components` como `object` y lo expandimos con `${{ each ... }}`.

### Contrato minimo sugerido para `components[]`
Campos recomendados por componente:
- `name`
- `component` (opcional; fallback funcional de `name`)
- `workingDirectory`
- `imageRepository`
- `dockerfilePath`
- `dockerBuildContext`
- `acaName`

Campos opcionales habituales (segun necesidad):
- `enabled`
- `deployEnabled`
- `envVars`
- `healthPath`
- `deployMode`

### Nota operativa sobre UX y seguridad
- Si `components` va en `parameters:` del root, aparece editable en "Run pipeline".
- Si se quiere evitar esa edicion manual, mantener `components` fuera de runtime parameters del root.
- El patron recomendado de esta PoC mantiene `components` visible en codigo, pero no expuesto como parametro editable de UX.

## 14) Ejemplos de consumo por nivel (detalle)

### A nivel de stage
```yaml
stages:
  - ${{ each component in parameters.components }}:
    - stage: ${{ format('Deploy_{0}', coalesce(component.name, component.component, 'default')) }}
      displayName: ${{ format('Deploy {0}', component.name) }}
      jobs:
        - job: Deploy
          steps:
            - script: echo "Deploy de ${{ component.name }}"
```

### A nivel de job
```yaml
stages:
  - stage: BuildAndPush
    jobs:
      - ${{ each component in parameters.components }}:
        - job: ${{ format('Build_{0}', coalesce(component.name, component.component, 'default')) }}
          displayName: ${{ format('Build {0}', component.name) }}
          steps:
            - script: echo "Build de ${{ component.imageRepository }}"
```

### A nivel de task (dentro de un job)
```yaml
stages:
  - stage: Validation
    jobs:
      - job: ValidateCatalog
        steps:
          - ${{ each component in parameters.components }}:
            - task: PowerShell@2
              displayName: ${{ format('Validate {0}', component.name) }}
              inputs:
                pwsh: true
                targetType: inline
                script: |
                  Write-Host "name=${{ component.name }}"
                  Write-Host "component=${{ component.component }}"
                  Write-Host "acaName=${{ component.acaName }}"
```

## 15) Reglas de resolucion de rutas Docker
En esta PoC, el build resuelve rutas Docker por componente con reglas estables:

- `workingDirectory`
  - Base funcional del componente.
  - Si es relativo, se interpreta respecto a `$(Build.SourcesDirectory)`.

- `dockerBuildContext`
  - Si viene vacio, usa `workingDirectory`.
  - Si es relativo, se interpreta respecto a `workingDirectory` resuelto (si existe), y si no respecto a `$(Build.SourcesDirectory)`.
  - Si es absoluto, se usa tal cual.

- `dockerfilePath`
  - Si viene vacio, usa `Dockerfile` dentro del `dockerBuildContext` resuelto.
  - Si es relativo, primero se intenta respecto a `workingDirectory` resuelto; si no existe, se resuelve respecto al `dockerBuildContext` resuelto.
  - Si es absoluto, se usa tal cual.

Ademas, antes de ejecutar `Docker@2 build`, se valida que existan:
- contexto Docker resuelto;
- ruta de Dockerfile resuelta.

Ejemplo minimo:

```yaml
components:
  - component: app1
    workingDirectory: src/services/dummy-test-app-1
    imageRepository: dummy/mi-proyecto-test-app-1
    dockerBuildContext: ''   # usa workingDirectory
    dockerfilePath: ''       # usa <dockerBuildContext>/Dockerfile
```


