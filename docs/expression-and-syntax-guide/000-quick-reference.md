# Índice Rápido de Referencia

Usa este índice para encontrar rápidamente la información que necesitas.

## Por Problema

### "Unrecognized value en parámetro de plantilla"
- **Causa**: Usaste `$( )` o variable en un parámetro con `values:`
- **Leer**: [02-template-parameter-validation.md](02-template-parameter-validation.md)
- **Solución**: Usa `${{ }}` en lugar de macro substitution

### "Variable no se resuelve donde la necesito"
- **Leer**: [01-expression-types.md](01-expression-types.md)
- **Tabla rápida**: Sección "Tabla de Referencia Rápida"

### "No entiendo cuándo se resuelve cada expresión"
- **Leer**: [03-yaml-processing-order.md](03-yaml-processing-order.md)
- **Diagrama**: Sección "Flujo General"

### "Necesito un patrón para mi caso específico"
- **Leer**: [04-common-use-cases.md](04-common-use-cases.md)
- **Checklist**: Al final del documento

---

## Por Expresión

### `${{ ... }}`
- Resuelve en: **Compile-time**
- Acceso a: `parameters`, funciones de expresión
- Mejor para: Parámetros con restricciones, lógica condicional compile-time
- **Leer**: [01-expression-types.md#1-expresiones-compile-time](01-expression-types.md)

### `$[ ... ]`
- Resuelve en: **Runtime**
- Acceso a: `variables`, salidas de trabajos, funciones runtime
- Mejor para: Derivar valores de variables en tiempo de ejecución
- **Leer**: [01-expression-types.md#2-expresiones-runtime](01-expression-types.md)

### `$( )`
- Resuelve en: **Después de compile (macro substitution)**
- Acceso a: Variables ya resueltas
- Mejor para: Usar en tareas y scripts
- **Leer**: [01-expression-types.md#3-macro-substitution](01-expression-types.md)

---

## Por Sección del Pipeline

### Parámetros
```
parameters:
  - name: MyParam
    type: string
    values: [...]        # ❌ No dinámico
    default: auto        # ❌ No dinámico
```

### Variables (Nivel Raíz)
```
variables:
  - name: MyVar          # ✅ Puede usar ${{ }} compile-time
    value: ${{ ... }}    # ✅ Puede usar $[ ] runtime
```

### Stages/Jobs (Condicionales)
```
- ${{ if ... }}:         # ✅ Lógica compile-time
    - stage: MyStage
      condition: |       # ✅ Condición compile-time
```

### Tasks/Steps
```
- task: SomeTask@1
  inputs:
    param: $(MyVar)      # ✅ Macro substitution en runtime
```

---

## Decisión Rápida: ¿Qué Expresión Usar?

```
¿Dónde la necesitas?
│
├─ ¿En parámetro de plantilla con 'values:'?
│  └─ → Usa ${{ }}
│
├─ ¿Lógica en compile-time (include/exclude)?
│  └─ → Usa ${{ if ... }}
│
├─ ¿Variable derivada de otras variables?
│  └─ ¿Necesita acceder a Build.*, variables?
│     └─ Sí → Usa $[ ] (runtime)
│     └─ No → Usa ${{ }} (compile-time)
│
├─ ¿En una tarea/script?
│  └─ → Usa $( )
│
└─ ¿No sabes?
   └─ → Lee [03-yaml-processing-order.md](03-yaml-processing-order.md)
```

---

## Resumen de Reglas de Oro

1. **La validación de parámetros ocurre en compile-time, antes de macro substitution**
   - Usa `${{ }}` para parámetros con restricciones

2. **Las variables no están disponibles en compile-time**
   - No puedes acceder a `variables.*` en `${{ }}`

3. **Para derivarvalores de variables, usa runtime**
   - `$[ variables['X'] ]` en variables
   - `$(X)` en tareas

4. **Confía en las plantillas para lógica compleja**
   - Las plantillas pueden manejar lógica que no cabe en un parámetro

5. **Cada contexto tiene sus limitaciones**
   - Compile-time: Solo parameters
   - Runtime: Todo, pero sin parámetros

---

## Documentación Oficial

- [Azure Pipelines YAML schema reference](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema)
- [Expressions in Azure Pipelines](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/expressions)
- [Template syntax reference](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema/template-reference)

---

## Contribución

Si encuentras un nuevo caso de uso o patrón no documentado aquí, considera agregarlo a esta guía.

**Última actualización**: Abril 14, 2026
**Versión**: 1.0
