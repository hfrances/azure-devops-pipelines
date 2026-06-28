# Guía de Expresiones y Sintaxis en Azure Pipelines

Esta carpeta contiene documentación detallada sobre cómo funcionan las expresiones, variables y parámetros en Azure DevOps Pipelines.

## Contenidos

- **[Tipos de Expresiones](01-expression-types.md)**: Explicación de `${{ }}`, `$[ ]` y `$( )`
- **[Validación de Parámetros en Plantillas](02-template-parameter-validation.md)**: Cómo y cuándo se validan los parámetros
- **[Orden de Procesamiento del YAML](03-yaml-processing-order.md)**: El flujo completo de cómo Azure DevOps procesa un pipeline
- **[Casos de Uso Comunes](04-common-use-cases.md)**: Patrones recomendados y antipatrones
- **[Component Catalog PoC](component-catalog-poc.md)**: Patron de catalogo de componentes en root + templates de stages/jobs
- **[Component Catalog Current State](component-catalog-current-state.md)**: Estado real de implementacion y migracion por repositorio
- **[Component Catalog Variable Patterns](component-catalog-variable-patterns.md)**: Convenciones de `variables:` por grupos (`Docker`, `Platform`, `Container private/public`)

## Contexto

Estos documentos fueron creados después de investigar problemas comunes con expresiones en parámetros de plantillas, especialmente cuando se intenta pasar lógica condicional o valores derivados a parámetros con restricciones (`values:`).

## Regla de Oro

**Si necesitas pasar un valor a un parámetro de plantilla que tiene restricciones (`values:`), usa una expresión compile-time (`${{ }}`), no variables con macro substitution (`$()`), porque la validación ocurre en un momento donde las macros aún no se han resuelto.**
