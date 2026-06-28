# Tipos de Expresiones en Azure DevOps Pipelines

Azure DevOps Pipelines proporciona tres tipos principales de expresiones para resolver valores dinámicamente. Es crítico entender cuándo se resuelve cada una.

## 1. Expresiones Compile-Time: `${{ }}`

### Características
- Se resuelven **durante la compilación del pipeline** (antes de la ejecución)
- El resultado se incrusta directamente en el YAML
- Pueden acceder a: `parameters`, funciones de expresión, literales

### Contexto disponible
```yaml
${{ if(...) }}          # Acceso a parameters, funciones if/eq/or/and/etc
${{ parameters.Deploy }}
${{ parameters.Environment }}
```

### Limitaciones
- **NO tiene acceso a variables** (que no se definen hasta después del compile-time)
- No puede acceder a `variables['XYZ']`
- No puede acceder a variables del sistema como `Build.SourceBranchName` (aunque algunos parecen disponibles)

### Casos de uso
```yaml
# ✅ Correcto: Pasar parámetro a plantilla
deploy: ${{ parameters.Deploy }}

# ✅ Correcto: Lógica condicional
- ${{ if eq(parameters.BuildMode, 'release') }}:
    - template: build-release.yml

# ✅ Correcto: Incluir/excluir trabajos basado en parámetro
jobs:
  - ${{ if eq(parameters.RunTests, true) }}:
    - job: RunTests
```

---

## 2. Expresiones Runtime: `$[ ]`

### Características
- Se resuelven **durante la ejecución del pipeline**
- Pueden acceder a variables del sistema, variables de nivel raíz y valores de pasos anteriores
- Se evalúan después de que la compilación haya completado

### Contexto disponible
```yaml
$[ variables['BuildConfiguration'] ]
$[ variables['Build.SourceBranchName'] ]
$[ stageDependencies.BuildStage.BuildJob.outputs['TaskName.VariableName'] ]
```

### Limitaciones
- No pueden acceder a `parameters` en este punto
- Se evalúan **después** de la validación de parámetros de plantillas

### Casos de uso
```yaml
# ✅ Correcto: Definir variable basada en lógica runtime
variables:
  ExternalFeedValue: $[iif(ne(variables['ExternalFeed'], ''), variables['ExternalFeed'], '')]

# ✅ Correcto: Acceder a salidas de trabajos anteriores
variables:
  ImageTag: $[ stageDependencies.BuildAndPush.BuildApi.outputs['ExportImageMetadata.ImageTag'] ]

# ✅ Correcto: Usar en tareas (ya durante la ejecución)
- script: echo "Branch is: $[ variables['Build.SourceBranchName'] ]"
```

---

## 3. Macro Substitution: `$( )`

### Características
- Sustituyen variables que **ya han sido resueltas** previamente
- Se usan principalmente en tareas y en línea de comandos
- El valor se conoce en el momento de ejecución

### Contexto disponible
```yaml
$(variableName)
$(Build.SourcesDirectory)
$(System.DefaultWorkingDirectory)
```

### Limitaciones
- Se resuelven **después** de la validación de parámetros de plantillas
- No pueden actuar como "puente" para pasar valores a parámetros con restricciones

### Casos de uso
```yaml
# ✅ Correcto: Usar en tareas
- script: echo "Building in $(Build.SourcesDirectory)"

# ✅ Correcto: Pasar variable a plantilla (si la plantilla no valida el parámetro)
- template: generic-template.yml
  parameters:
    workingDir: $(Build.SourcesDirectory)
```

---

## Comparativa: Cuándo se resuelve cada una

```
Azure DevOps Compilation & Execution Flow:
═══════════════════════════════════════════════════════════════════════

COMPILE-TIME (Parse YAML):
1. Parsear estructura YAML
2. ✅ Expresiones ${{ }} se resuelven AQUÍ
3. Validar parámetros de plantillas (busca valores en lista permitida)
   ↳ En este punto, $( ) aún no se ha resuelto
   ↳ Las variables aún no están disponibles
4. ✅ Macro substitution $( ) ocurre aquí
5. Incluir plantillas externas

RUNTIME (Execute Pipeline):
6. Procesar variables
7. ✅ Expresiones $[ ] se resuelven AQUÍ
8. Ejecutar trabajos y tareas
9. ✅ Macro substitution $( ) se resuelve nuevamente con valores actualizados
```

---

## Tabla de Referencia Rápida

| Expresión | Tipo | Cuándo se resuelve | Acceso a `parameters` | Acceso a `variables` | Acceso a `Build.*` | Uso en parámetros de plantilla |
|-----------|------|-------------------|----------------------|----------------------|-------------------|-------------------------------|
| `${{ }}` | Compile-time | Compilación | ✅ Sí | ❌ No | ⚠️ Limitado | ✅ **Sí (recomendado)** |
| `$[ ]` | Runtime | Ejecución | ❌ No | ✅ Sí | ✅ Sí | ❌ No (demasiado tarde) |
| `$( )` | Macro | Después de compile | ❌ No | ✅ Sí (después) | ✅ Sí | ❌ No (demasiado tarde) |

---

## Resumen de Recomendaciones

1. **Para parámetros de plantilla con restricciones**: Usa **siempre** `${{ }}`
2. **Para lógica condicional en compile-time**: Usa `${{ if(...) }}`
3. **Para acceder a variables en compile-time**: No es posible, necesitas workarounds
4. **Para acceder a variables en runtime**: Usa `$[ ]` en variables, `$( )` en tareas
5. **Para salidas de trabajos anteriores**: Usa `$[ stageDependencies... ]`
