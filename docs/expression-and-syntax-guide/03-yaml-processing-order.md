# Orden de Procesamiento del YAML en Azure DevOps Pipelines

Para entender por qué ciertas expresiones funcionan en algunos lugares y no en otros, es crítico comprender el orden exacto en que Azure DevOps procesa el YAML de tu pipeline.

## Flujo General: Compile-Time vs Runtime

```
┌─────────────────────────────────────────────────────────────────┐
│                   Azure DevOps Pipeline Processing               │
└─────────────────────────────────────────────────────────────────┘

FASE 1: COMPILE-TIME (Parse and Validate YAML)
══════════════════════════════════════════════════════════════════

1. [PARSE] Leer el archivo azure-pipelines.yml
   └─ Acceso a: Estructura YAML bruta

2. [PARSE] Identificar secciones: trigger, parameters, variables, stages, jobs
   └─ Acceso a: Estructura definida

3. [EXPRESSION RESOLVE] Resolver expresiones ${{ }} 
   📍 CONTEXTO DISPONIBLE:
      ✅ parameters.*   (valores de parámetros)
      ✅ Funciones: if(), eq(), and(), or(), ne(), startsWith(), etc.
      ✅ Literales: 'true', 'false', strings
      ❌ variables.*    (aún no procesadas)
      ❌ Build.SourceBranch (aún no disponible)

4. [TEMPLATE INCLUDE] Incluir plantillas externas
   └─ Pasar parámetros (que se validarán ahora)

5. [VALIDATE] Validar parámetros de plantillas
   📍 VALIDACIÓN DE PARÁMETROS CON RESTRICCIONES:
      - Buscar parámetros con 'values:'
      - Validar que el valor pasado esté en la lista
      ❌ $( ) aún NO se ha resuelto aquí
      ❌ Las variables NO están disponibles aún

6. [MACRO SUBSTITUTE] Resolver macro substitutions $( )
   📍 Sustituir $(variableName) pero:
      ❌ Las variables aún no tienen valores (se procesarán en runtime)
      ⚠️ Sustitución ocurre, pero con valores compilados

7. [FINALIZE] Generar YAML final compilado

════════════════════════════════════════════════════════════════════

FASE 2: RUNTIME (Execute Pipeline)
══════════════════════════════════════════════════════════════════

8. [PROCESS VARIABLES] Procesar sección 'variables:'
   📍 CONTEXTO DISPONIBLE:
      ✅ Expresiones $[ ]
      ✅ Funciones: iif(), variables.*
      ❌ parameters.* (no disponibles en runtime)

9. [EXECUTE] Procesar stages → jobs → tasks
   📍 CONTEXTO DISPONIBLE:
      ✅ Macro substitution $( )   (variables resueltas)
      ✅ Variables del sistema    (Build.*, System.*)
      ✅ Variables de trabajos anteriores

10. [COMPLETE] Pipeline completado
```

## Ejemplo Práctico: Dónde se Resuelve Cada Cosa

```yaml
# ═══════════════════════════════════════════════════════════════
# COMPILE-TIME
# ═══════════════════════════════════════════════════════════════

parameters:
  - name: Environment
    type: string
    values: 
      - dev
      - prod
    default: dev

# 1. ${{ }} se resuelve AQUÍ (compile-time)
variables:
  - name: BuildConfig
    value: ${{ if eq(parameters.Environment, 'prod') then 'Release' else 'Debug' }}
    # Resultado: 'Release' o 'Debug' (se conoce en compile-time)

# 2. Parámetro de plantilla se valida AQUÍ
- template: build.yml@repo
  parameters:
    environment: ${{ parameters.Environment }}  # ✅ Se valida
    buildConfig: $(BuildConfig)                 # ❌ Se intenta sustituir pero variable aún no existe

# ═══════════════════════════════════════════════════════════════
# RUNTIME (durant la ejecución)
# ═══════════════════════════════════════════════════════════════

jobs:
  - job: BuildJob
    variables:
      # 3. $[ ] se resuelve AQUÍ (runtime)
      - name: OutputPath
        value: $[ iif(eq(variables['BuildConfig'], 'Release'), '/release', '/debug') ]
    
    steps:
      # 4. $( ) se resuelve AQUÍ (runtime, en tareas)
      - script: echo "Building in $(OutputPath) with $(BuildConfig)"
```

## Sección por Sección: Dónde están disponibles las expresiones

### TRIGGER
```yaml
trigger:
  batch: true
  branches:
    include:
      - main
# ❌ No hay expresiones aquí, es configuración estática
```

### PARAMETERS
```yaml
parameters:
  - name: Deploy
    type: string
    values:      # ❌ No pueden ser dinámicas
      - auto
      - true
      - false
    default: auto
```

### VARIABLES (Nivel Raíz)
```yaml
variables:
  # COMPILE-TIME: ${{ }} se puede usar
  - name: ComputedVar
    value: ${{ if(...) ... }}  # ✅ Se resuelve en compile-time
  
  # RUNTIME: $[ ] se resuelve después
  - name: RuntimeVar  
    value: $[iif(...) ... ]    # ✅ Se resuelve en runtime
```

### STAGES
```yaml
stages:
  # COMPILE-TIME: ${{ }} disponible para incluir/excluir
  - ${{ if eq(parameters.BuildMode, 'full') }}:
    - stage: FullBuild
      displayName: Full Build Stage
      
      # La definición de stage es static
      jobs:
        - job: MyJob
```

### JOBS (dentro de stage)
```yaml
jobs:
  # COMPILE-TIME: ${{ }} para incluir/excluir
  - ${{ if eq(parameters.RunTests, true) }}:
    - job: Tests
      displayName: Run Tests
      
      # Variables de ESTE job (runtime)
      variables:
        - name: TestResults
          value: $[iif(...) ... ]  # ✅ Runtime
```

### STEPS (dentro de job)
```yaml
steps:
  # Las tareas siempre se ejecutan en RUNTIME
  - script: echo "Hello $(BuildConfig)"
    # $(BuildConfig) se resuelve en runtime
  
  - task: SomeTask@1
    inputs:
      path: $(Build.SourcesDirectory)  # Runtime
```

## Timeline Visual: El Viaje de una Variable

```
Escenario: Pasar valor derivado a parámetro de plantilla

┌─────────────────────────────────────────────────────────┐
│ 1. Usuario define variable en nivel raíz (YAML)          │
│    variables:                                             │
│      - name: effectiveDeploy                              │
│        value: ${{ if(...) ... }}                           │
└─────────────────────────────────────────────────────────┘
                        ↓ COMPILE-TIME: Paso 3
┌─────────────────────────────────────────────────────────┐
│ 2. Se resuelve efectiveDeploy → 'true' o 'false'        │
│    (resultado: cadena en el YAML compilado)              │
└─────────────────────────────────────────────────────────┘
                        ↓ COMPILE-TIME: Paso 6
┌─────────────────────────────────────────────────────────┐
│ 3. Se intenta pasar como parámetro: deploy: $(...)      │
│    PROBLEMA: $( ) no se ha resuelto aún                  │
│              Azure DevOps ve "$(effectiveDeploy)"        │
│              Valida si esto está en [auto,true,false]    │
│              ❌ NO lo encuentra → ERROR                  │
└─────────────────────────────────────────────────────────┘

SOLUCIÓN: Usar ${{ if(...) }} directamente:
            deploy: ${{ if(...) then 'true' else 'false' }}
            
            Ahora:
            - ${{ }} se resuelve en Paso 3 → 'true'
            - Se valida 'true' en Paso 5 ✅
```

## Checklist: ¿Dónde puedo usar cada expresión?

| Ubicación | ${{ }} | $[ ] | $( ) | Notas |
|-----------|--------|------|------|-------|
| `parameters.*.default:` | ✅ No necesario (estático) | ❌ | ❌ | Los defaults son estáticos |
| `variables.*.value:` | ✅ (compile) | ✅ (runtime) | ✅ (runtime) | Depende de cuándo lo necesites |
| Parámetro de plantilla | ✅ **Recomendado** | ❌ | ❌ | Validación ocurre en compile-time |
| Condición de stage/job | ✅ | ✅ | ❌ | Las condiciones son compile-time |
| Entrada de tarea | ✅ Para deriv. | ✅ | ✅ **Recomendado** | Runtime, sin validación |
| Script inline | ✅ | ✅ | ✅ | Runtime, el script ve todo |

## Reglas Mnemotécnicas

1. **Para parámetros con `values:`** → Siempre `${{ }}`
2. **Para acceder a variables en compile-time** → No es posible, usa parameters
3. **Para acceder a variables en runtime** → `$[ ]` en variables, `$( )` en scripts
4. **Para condiciones** → `${{ if... }}` en compile-time
5. **Para salidas de trabajos anteriores** → `$[ stageDependencies... ]`

