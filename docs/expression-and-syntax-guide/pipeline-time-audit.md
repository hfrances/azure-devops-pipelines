# Pipeline Time Audit

## Tabla de contenido
- [Resumen ejecutivo](#resumen-ejecutivo)
- [Evidencia observada](#evidencia-observada)
- [Diagnostico](#diagnostico)
- [Propuestas de mejora (sin cambios de codigo en este documento)](#propuestas-de-mejora-sin-cambios-de-codigo-en-este-documento)
- [Prioridad recomendada](#prioridad-recomendada)

## Resumen ejecutivo
La duracion total observada del pipeline completo ronda los **8 minutos**, lo cual es alto para el tamano del proyecto `dummy-test`.

Distribucion aproximada por stage (segun la evidencia capturada):
- `Preparation`: ~1m 45s
- `Build and push`: ~1m 20s
- `Deploy`: ~4m 54s

La mayor parte del tiempo se concentra en el stage de **Deploy**.

## Evidencia observada

### Preparation
- `Prepare app 1`: ~12s
- `Prepare app 2`: ~12s
- `Prepare summary`: ~3s
- El tiempo agregado del stage incluye overhead de orquestacion, inicializacion de jobs y publicacion de artefactos.

### Build and push
- `Build and push app 1`: ~34s
- `Build and push app 2`: ~33s
- Dentro de cada job, los pasos mas costosos son:
  - `Build image`: ~7s
  - `Push image`: ~12s

### Deploy
- `Deploy app 1`: ~2m 23s
- `Deploy app 2`: ~2m 18s
- En cada job de deploy, los pasos con mayor coste son:
  - `Prepare Azure CLI`: ~32s
  - `Deploy ACA`: ~1m 37s

## Diagnostico
1. El cuello de botella principal es **Deploy ACA**, no la construccion de imagenes.
2. Existe coste fijo repetido por cada componente desplegado (preparacion CLI + validaciones + despliegue).
3. El total percibido se incrementa por overhead de inicializacion/finalizacion de multiples jobs.
4. Para ciclos de prueba frecuentes, ejecutar siempre despliegue completo penaliza mucho el feedback.

## Propuestas de mejora (sin cambios de codigo en este documento)

### 1) Modo rapido para iteracion
- Usar ejecuciones con `Deploy=false` cuando el objetivo sea validar cambios de CI/plantillas.
- Mantener despliegue completo para ramas objetivo o ejecuciones manuales de validacion final.

### 2) Separar pipeline rapido vs pipeline completo

Objetivo: reducir el tiempo de feedback en iteracion diaria sin perder una validacion end-to-end cuando realmente hace falta.

Pipeline rapido (validacion tecnica):
- Ejecuta: `Preparation` + `Build` (y opcionalmente `Push`).
- No ejecuta: `Deploy`.
- Uso recomendado: commits frecuentes, ramas de trabajo, pruebas de cambios en templates/CI.
- Beneficio: elimina el mayor cuello de botella (deploy), bajando mucho el tiempo total por ejecucion.

Pipeline completo (validacion operativa):
- Ejecuta: `Preparation` + `Build and push` + `Deploy`.
- Uso recomendado: ramas objetivo (main/master/staging), validaciones manuales finales, releases.
- Beneficio: mantiene la garantia funcional completa cuando el cambio ya esta listo para promocion.

Formas de aplicarlo:
- Opcion A: una sola pipeline con parametro (`Deploy=true/false`) y politica de uso por rama.
- Opcion B: dos pipelines separadas (`fast` y `full`) con triggers distintos.

Ejemplo para `dummy-test`:
- `fast`: ejecutar con `Deploy=false` para validar en ~2-3 minutos.
- `full`: ejecutar con `Deploy=true` solo cuando se necesite validacion real en ACA.

### 3) Revisar paralelismo disponible en Azure DevOps
- Confirmar si la organizacion/proyecto dispone de mas de 1 parallel job.
- Si solo hay 1, parte de la ejecucion se serializa y aumenta el tiempo final.

### 4) Optimizacion de despliegue por componente
- El mayor potencial esta en reducir coste repetido de deploy por app.
- Area candidata futura: consolidar pasos comunes previos al deploy o replantear estrategia de despliegue (siempre evaluando riesgo operativo).

### 5) Medicion continua
- Añadir registro historico de duracion por stage/job para comparar mejoras.
- Definir objetivo de tiempo para modo rapido (por ejemplo, <3 minutos) y modo completo.

### Alternativa remota: patron de bootstrap minimo (no prioritaria)
Como opcion avanzada, podria introducirse un job inicial muy corto (bootstrap) para forzar que un flujo "preferente" arranque antes que el resto, manteniendo luego paralelismo en los demas jobs.

Importante:
- No es una prioridad ahora mismo.
- No se propone implementarlo en esta fase.
- Se considera solo como alternativa futura si el orden de arranque en paralelo pasa a ser un requisito operativo.

## Prioridad recomendada
1. Activar modo rapido (`Deploy=false`) para pruebas de iteracion.
2. Validar paralelismo real de agentes en la organizacion.
3. Diseñar version de pipeline rapido dedicado.
4. Evaluar refactor del stage Deploy si sigue siendo el cuello de botella dominante.

