# Validación de Parámetros en Plantillas

Uno de los conceptos más críticos (y fuente de errores comunes) en Azure DevOps Pipelines es entender cuándo y cómo se validan los parámetros de las plantillas.

## Validación Básica

### Parámetros con restricciones de valores

Una plantilla puede definir restricciones en los valores permitidos para un parámetro:

```yaml
# Plantilla: deploy-stage-template.yml
parameters:
  - name: deploy
    type: string
    values:
      - auto
      - true
      - false
    default: auto
```

Cuando se incluye esta plantilla, Azure DevOps **valida** que el valor pasado sea uno de los permitidos:

```yaml
# Pipeline que usa la plantilla
- template: deploy-stage-template.yml@templates-repo
  parameters:
    deploy: true  # ✅ Válido
    # deploy: maybe  # ❌ Error: no está en la lista de valores permitidos
```

## Cuándo Ocurre la Validación

**La validación ocurre en TIEMPO DE COMPILACIÓN, antes de que se resuelvan las macro substitutions.**

### Momento exacto en el flujo de compilación

```
1. Parsear YAML del pipeline
2. Resolver expresiones ${{ }} (compile-time)
3. ← AQUÍ COMIENZA LA VALIDACIÓN DE PARÁMETROS
4. Validar que los valores de parámetros estén en la lista permitida
5. Resolver macro substitutions $( ) ← Demasiado tarde para validación
6. Incluir la plantilla
```

## Problema Común: Usar `$( )` en Parámetros con Restricciones

### ❌ INCORRECTO - Falla con "Unrecognized value"

```yaml
variables:
  - name: effectiveDeploy
    value: ${{ if(...condition...) then 'true' else 'false' }}

- template: deploy-stage-template.yml@templates-repo
  parameters:
    deploy: $(effectiveDeploy)  # ❌ Error
```

**Error recibido:**
```
Unrecognized value: 'auto'. Located at position 1 within expression: '$(effectiveDeploy)'.
```

**Por qué falla:**
1. Azure DevOps ve el parámetro `deploy: $(effectiveDeploy)`
2. Intenta validar que `$(effectiveDeploy)` esté en `[auto, true, false]`
3. En este punto, `$(effectiveDeploy)` aún es una cadena literal, no se ha resuelto
4. La cadena `"$(effectiveDeploy)"` no está en la lista → Error de validación

### ✅ CORRECTO - Usar `${{ }}`

```yaml
- template: deploy-stage-template.yml@templates-repo
  parameters:
    deploy: ${{ parameters.Deploy }}  # ✅ Funciona
```

**Por qué funciona:**
1. `${{ parameters.Deploy }}` es una expresión compile-time
2. Se resuelve inmediatamente a `'auto'`, `'true'` o `'false'`
3. Azure DevOps valida el valor resuelto (e.j., `'true'`)
4. El valor está en la lista → Validación pasa ✅

## Variables a Nivel Raíz: Caso Especial

Existe un patrón que **sí funciona** incluso aunque parezca similar a lo anterior:

```yaml
# ✅ FUNCIONA - Variable a nivel raíz usada en tareas
variables:
  ExternalFeedValue: $[iif(ne(variables['ExternalFeed'], ''), variables['ExternalFeed'], '')]
  
# Y luego se usa en una tarea:
- task: DotNetCoreCLI@2
  inputs:
    externalFeedCredentials: $(ExternalFeedValue)
```

**Por qué funciona aquí:**
- `ExternalFeedValue` **no es un parámetro de plantilla**, es una variable del pipeline
- Se define a nivel raíz y se usa dentro del mismo pipeline
- La validación de parámetros de plantilla **no aplica** aquí
- Cuando se ejecuta la tarea, `$(ExternalFeedValue)` ya se ha resuelto

## Validación de Otros Tipos de Parámetros

### Parámetros sin restricciones
```yaml
parameters:
  - name: customValue
    type: string
    # Sin 'values:', por lo que acepta cualquier string
    default: ''

# Puedes pasar cualquier valor:
- template: build.yml@templates-repo
  parameters:
    customValue: $(Build.SourcesDirectory)  # ✅ Funciona, sin validación
```

### Parámetros booleanos
```yaml
parameters:
  - name: runTests
    type: boolean
    default: true

# Los booleanos no tienen restricción de valores:
- template: build.yml@templates-repo
  parameters:
    runTests: ${{ eq(parameters.Environment, 'production') }}  # ✅ Funciona
```

### Parámetros de objeto/lista
```yaml
parameters:
  - name: variables
    type: object
    default: {}

# Sin restricción de valores:
- template: build.yml@templates-repo
  parameters:
    variables: ${{ parameters.CustomVars }}  # ✅ Funciona
```

## Casos de Uso Recomendados

### 1. Pasar el parámetro tal cual
```yaml
- template: deploy-stage-template.yml@repo
  parameters:
    deploy: ${{ parameters.Deploy }}
```

### 2. Lógica condicional para incluir/excluir plantillas
```yaml
- ${{ if eq(parameters.BuildMode, 'release') }}:
  - template: build-release.yml

- ${{ if ne(parameters.BuildMode, 'release') }}:
  - template: build-debug.yml
```

### 3. Determinar parámetro basado en ramas (compile-time)
⚠️ **Limitación**: No puedes acceder a `Build.SourceBranchName` en compile-time
- Necesitas pasarlo como parámetro
- O manejar la lógica dentro de la plantilla

### 4. Variables para uso en tareas (runtime)
```yaml
variables:
  - name: imageTag
    value: $[ stageDependencies.BuildAndPush.BuildApi.outputs['ExportImageMetadata.ImageTag'] ]
    
# Usar en tarea:
- script: docker run $(imageTag)
```

## Antipatrones

### ❌ NO HAGAS ESTO

```yaml
# Intenta usar macro substitution en parámetro con restricciones
- template: deploy.yml@repo
  parameters:
    deploy: $(derivedValue)  # Falla en validación

# Intenta usar $[ ] en parámetro de plantilla
- template: deploy.yml@repo
  parameters:
    deploy: $[ if(eq(variables['Build.SourceBranch'], 'refs/heads/main'), 'true', 'false') ]
    # Falla: $[ ] no se resuelve a tiempo

# Intenta acceder a variables en ${{ }}
deploy: ${{ variables.MyVar }}  # Falla: variables no disponible en compile-time
```

## Resumen

| Escenario | Solución | Razón |
|-----------|----------|-------|
| Parámetro con `values:` en plantilla | Usa `${{ }}` | Se resuelve antes de validación |
| Variable para usar en tarea | Usa `$[ ]` o `$( )` | Runtime, sin validación |
| Lógica condicional compile-time | Usa `${{ if(...) }}` | Se resuelve en compile-time |
| Derivar valor desde variable | No es posible en `${{ }}` | Variables no disponibles en compile-time |

