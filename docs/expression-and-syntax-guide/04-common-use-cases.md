# Casos de Uso Comunes y Patrones Recomendados

Esta sección contiene patrones reales, soluciones probadas y antipatrones a evitar.

## 1. Pasar Parámetro Tal Cual a Plantilla

### Caso de Uso
Tienes un parámetro en tu pipeline que necesitas pasar a una plantilla con restricciones de valores.

### ✅ CORRECTO

```yaml
# Parámetro en nivel raíz
parameters:
  - name: Deploy
    displayName: Deploy
    type: string
    values:
      - auto
      - true
      - false
    default: auto

# Pasar a plantilla
- template: stages-templates/deploy-stage-template-stage.yml@hf-azure-pipelines-repo
  parameters:
    deploy: ${{ parameters.Deploy }}
```

### ❌ INCORRECTO

```yaml
# Intentar pasar como macro substitution
- template: stages-templates/deploy-stage-template-stage.yml@hf-azure-pipelines-repo
  parameters:
    deploy: $(Deploy)  # ❌ No funciona, $(Deploy) no existe
```

---

## 2. Condicional: Incluir/Excluir Plantilla Basado en Parámetro

### Caso de Uso
Según un parámetro, incluir una plantilla u otra.

### ✅ CORRECTO

```yaml
parameters:
  - name: BuildMode
    type: string
    values:
      - debug
      - release
    default: debug

- ${{ if eq(parameters.BuildMode, 'release') }}:
  - template: build-release.yml@repo

- ${{ if eq(parameters.BuildMode, 'debug') }}:
  - template: build-debug.yml@repo
```

### Alternativa más limpia

```yaml
- ${{ if ne(parameters.BuildMode, 'release') }}:
  - template: build-debug.yml@repo
- ${{ if eq(parameters.BuildMode, 'release') }}:
  - template: build-release.yml@repo
```

---

## 3. Usar Variable de Runtime en Tareas

### Caso de Uso
Necesitas una variable que se calculate en runtime y usarla en tareas.

### ✅ CORRECTO

```yaml
variables:
  # Esta variable se calcula en runtime
  - name: OutputDirectory
    value: $[iif(eq(variables['Build.SourceBranch'], 'refs/heads/main'), '$(Build.ArtifactStagingDirectory)/prod', '$(Build.ArtifactStagingDirectory)/dev')]

jobs:
  - job: BuildJob
    steps:
      - script: echo "Output path: $(OutputDirectory)"
      - task: PublishPipelineArtifact@1
        inputs:
          targetPath: $(OutputDirectory)
          artifact: 'build-output'
```

**Por qué funciona:**
- `$[iif(...)]` se resuelve durante runtime
- `$(OutputDirectory)` se resuelve cuando la tarea se ejecuta
- No hay validación de parámetros involucrada

---

## 4. Acceder a Salida de Trabajo Anterior

### Caso de Uso
Un trabajo (BuildApi) genera un nombre de imagen. Un trabajo posterior (DeployApi) necesita usarlo.

### ✅ CORRECTO

```yaml
stages:
  - stage: BuildStage
    jobs:
      - job: BuildApi
        steps:
          - script: |
              $imageTag = "myapp-$(Build.BuildId)"
              echo "##vso[task.setvariable variable=ImageTag;isOutput=true]$imageTag"
            name: BuildOutput
        
  - stage: DeployStage
    dependsOn: BuildStage
    variables:
      # Acceder a salida del trabajo anterior
      ImageTag: $[ stageDependencies.BuildStage.BuildApi.outputs['BuildOutput.ImageTag'] ]
    jobs:
      - job: DeployApi
        steps:
          - script: echo "Deploying $(ImageTag)"
```

**Por qué funciona:**
- Las salidas de trabajos están disponibles en runtime
- Se usan con sintaxis `$[ stageDependencies... ]`
- Se pueden asignar a variables para usarlas en tareas

---

## 5. Parámetro Común pero Multiestado

### Caso de Uso
Necesitas que un parámetro afecte el comportamiento en múltiples lugares, pero sin duplicar lógica.

### ✅ CORRECTO - Pasar a Plantilla (Recomendado)

```yaml
parameters:
  - name: Environment
    type: string
    values:
      - development
      - staging
      - production
    default: development

stages:
  - template: deploy-stage.yml@repo
    parameters:
      environment: ${{ parameters.Environment }}
      
  - template: smoke-tests-stage.yml@repo
    parameters:
      environment: ${{ parameters.Environment }}
      targetServerUrl: ${{ if eq(parameters.Environment, 'production') then 'https://api.prod.example.com' else 'https://api.staging.example.com' }}
```

**Ventajas:**
- La plantilla maneja toda la lógica
- No duplicas decisiones
- Es testeable

---

## 6. Variable Basada en Rama (IMPOSIBLE en Compile-Time)

### Caso de Uso
Quieres definir comportamiento diferente según la rama (main vs feature branches).

### ❌ INCORRECTO - Intenta usar Build.SourceBranch en compile-time

```yaml
variables:
  - name: DeployToProduction
    value: ${{ if startsWith(variables['Build.SourceBranchName'], 'refs/heads/main') then 'true' else 'false' }}
    # ❌ Error: variables no disponibles en compile-time
```

### ✅ CORRECTO - Usar en Runtime o en Plantilla

**Opción 1: Runtime variable**
```yaml
variables:
  - name: IsMainBranch
    value: $[ eq(variables['Build.SourceBranch'], 'refs/heads/main') ]

stages:
  - stage: MaybeDeployStage
    condition: eq(variables['IsMainBranch'], true)
    jobs:
      - job: DeployProduction
```

**Opción 2: Lógica en Plantilla (Recomendado)**
```yaml
# La plantilla contiene la lógica
- template: conditional-deploy.yml@repo
  parameters:
    deploymentType: ${{ parameters.DeploymentType }}
    
# Y dentro de la plantilla:
# - stage: ${{ if eq(variables['Build.SourceBranch'], 'refs/heads/main') }}:
#     # Deploy logic
```

---

## 7. Parámetro Derivado (Workaround)

### Caso de Uso
Necesitas un "parámetro derivado" sin poder hacer lógica compleja en parámetros.

### Patrón: Pasar Parámetro Base + Manejar en Plantilla

```yaml
# Pipeline principal
parameters:
  - name: Variant
    type: string
    values:
      - debug
      - release
    default: debug

- template: build.yml@repo
  parameters:
    variant: ${{ parameters.Variant }}

# En la plantilla (build.yml):
parameters:
  - name: variant
    type: string

variables:
  - name: BuildConfig
    value: ${{ if eq(parameters.variant, 'release') then 'Release' else 'Debug' }}

jobs:
  - job: Build
    steps:
      - script: dotnet build /p:Configuration=$(BuildConfig)
```

**Ventaja:**
- La plantilla maneja la derivación
- No necesitas lógica complicada en el pipeline principal

---

## 8. Parámetros Opcionales Condicionalmente

### Caso de Uso
Ciertos parámetros solo son relevantes en ciertos contextos.

### ✅ CORRECTO

```yaml
parameters:
  - name: BuildMode
    type: string
    values:
      - standard
      - custom
    default: standard
  - name: CustomTarget
    type: string
    default: ''

stages:
  - ${{ if eq(parameters.BuildMode, 'standard') }}:
    - template: standard-build.yml@repo
  
  - ${{ if eq(parameters.BuildMode, 'custom') }}:
    - template: custom-build.yml@repo
      parameters:
        target: ${{ parameters.CustomTarget }}
```

---

## 9. ExternalFeedCredentials Pattern

### Caso de Uso
Necesitas usar credenciales externas de Nuget condicionalmente, pero la validación ocurre en compile-time.

### ✅ CORRECTO - (Inspirado en HMY.IA.ImageCompare)

```yaml
variables:
  # Variable que se resuelve con lógica, usada en runtime
  ExternalFeedValue: $[iif(ne(variables['ExternalFeed'], ''), variables['ExternalFeed'], '')]

jobs:
  - job: RestorePackages
    steps:
      - task: DotNetCoreCLI@2
        displayName: Restore
        inputs:
          command: restore
          projects: '**/*.csproj'
          externalFeedCredentials: $(ExternalFeedValue)  # Se resuelve aquí
```

**Por qué funciona:**
- No es un parámetro de plantilla, es una variable de nivel raíz
- Se resuelve durante runtime, cuando la tarea se ejecuta
- No hay restricción de valores

---

## 10. Matrix Build (Múltiples Configuraciones)

### Caso de Uso
Ejecutar el mismo job con diferentes configuraciones.

### ✅ CORRECTO

```yaml
parameters:
  - name: Configurations
    type: object
    default:
      - name: Debug
        flag: debug
      - name: Release  
        flag: release

jobs:
  - ${{ each config in parameters.Configurations }}:
    - job: Build_${{ config.name }}
      displayName: Build ${{ config.name }}
      steps:
        - script: dotnet build /p:Configuration=${{ config.flag }}
```

---

## Antipatrones Comunes

### ❌ 1. Macro Substitution en Parámetro con Restricciones

```yaml
variables:
  - name: MyValue
    value: true

# ❌ Esto falla
- template: deploy.yml@repo
  parameters:
    deploy: $(MyValue)
    # Error: "$(MyValue)" no está en [auto, true, false]
```

**Solución**: Usa `${{ }}` o manejar en la plantilla

---

### ❌ 2. Variables en Expresiones Compile-Time

```yaml
# ❌ Esto no funciona
deploy: ${{ if eq(variables['Build.SourceBranch'], 'main') then 'true' else 'false' }}
# Error: variables no disponibles en compile-time
```

**Solución**: Usa runtime variables o inline en tareas

---

### ❌ 3. Acceder a Parámetros en Runtime

```yaml
jobs:
  - job: MyJob
    steps:
      - script: echo "$(parameters.Environment)"  # ❌ params no disponibles en runtime
```

**Solución**: Guarda como variable de nivel raíz o en job

---

### ❌ 4. Condicionales en Valores de Parámetro

```yaml
parameters:
  - name: Values
    type: object
    values:  # ❌ No pueden ser dinámicas
      - ${{ if ... }}:  # Esto no funciona
        - option1
```

**Solución**: Usa parámetro sin restricciones o múltiples parámetros

---

## Checklist de Solución de Problemas

Si tu expresión no funciona, prueba este checklist:

1. ¿Es un parámetro de plantilla con `values:`?
   - SÍ → Usa `${{ }}`
   - NO → Continúa

2. ¿Necesito compile-time behavior?
   - SÍ → Usa `${{ }}`, accede solo a `parameters`
   - NO → Continúa

3. ¿Es una tarea que se ejecuta en runtime?
   - SÍ → Usa `$( )` o `$[ ]`
   - NO → Continúa

4. ¿Necesito acceder a variables?
   - SÍ → Usa `$[ ]` (runtime) o define como variable de nivel raíz
   - NO → ✅ Debe funcionar

5. ¿Es una salida de trabajo anterior?
   - SÍ → Usa `$[ stageDependencies... ]`

