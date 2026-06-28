# Logging Commands: El "Control Remoto" del Pipeline (`##vso`)

Los **Logging Commands** son el protocolo de comunicación que permite a tus scripts (PowerShell, Bash, Python) "hablar" directamente con el Agente de Azure Pipelines. Es la forma en que un proceso aislado le ordena al servidor que cambie variables, suba archivos o pinte logs de colores.

## Tabla de contenidos

- [Gestión de Variables: task.setvariable](#gestión-de-variables-tasksetvariable)
- [Visibilidad y Feedback: task.issue y task.debug](#visibilidad-y-feedback-taskissue-y-taskdebug)
- [Control de Estado y Progreso: task.setprogress y task.complete](#control-de-estado-y-progreso-tasksetprogress-y-taskcomplete)
- [Identidad del Pipeline: build.updatebuildnumber y build.addbuildtag](#identidad-del-pipeline-buildupdatebuildnumber-y-buildaddbuildtag)
- [Reportes y Evidencias: task.upload* y task.addattachment](#reportes-y-evidencias-taskupload-y-taskaddattachment)
- [Conexiones y Telemetría: task.setendpoint y telemetry.publish](#conexiones-y-telemetría-tasksetendpoint-y-telemetrypublish)
- [Tabla de Referencia Rápida](#tabla-de-referencia-rápida)
- [Reglas de Oro para el uso de ##vso](#reglas-de-oro-para-el-uso-de-vso)

## 1. Gestión de Variables: `task.setvariable`

Es el comando más potente. Permite que un dato generado dinámicamente "viva" después de que el script termine para ser usado por otros pasos o trabajos.

### Casos de uso
```powershell
# Standard variable (available in subsequent steps of the same job)
Write-Host "##vso[task.setvariable variable=MyData]value"

# Output variable (available for OTHER JOBS or STAGES)
Write-Host "##vso[task.setvariable variable=FinalVersion;isOutput=true]1.5.0"

# Secret variable (automatically masked with *** in all logs)
Write-Host "##vso[task.setvariable variable=ApiToken;issecret=true]p@ssword123"

# Read-only variable (prevents subsequent scripts from modifying it)
Write-Host "##vso[task.setvariable variable=ImmutableVar;isreadonly=true]fixed_value"
```

---

## 2. Visibilidad y Feedback: `task.issue` y `task.debug`

Controlan cómo se reportan los incidentes y qué información técnica se muestra.

### Casos de uso
```powershell
# Debug: Blue text, only visible if 'system.debug' is true
Write-Host "##vso[task.debug]Internal variable state: $internalVar"

# Warning: Yellow text, appears in the pipeline summary
Write-Host "##vso[task.issue type=warning]Low disk space detected on agent."

# Error: Red text, highlights the issue in the summary and logs
Write-Host "##vso[task.issue type=error]Database connection failed after 3 retries."

# Advanced issue: Including source file and line info
Write-Host "##vso[task.issue type=error;sourcepath=main.cs;linenumber=42;columnnumber=5;code=ERR001;]Syntax error."
```

---

## 3. Control de Estado y Progreso: `task.setprogress` y `task.complete`

Ideales para tareas de larga duración o para forzar la finalización de un paso.

### Casos de uso
```powershell
# Show progress percentage in the UI
Write-Host "##vso[task.setprogress value=75;]Installing dependencies..."

# Force a step to finish with a specific result
# Results: Succeeded, SucceededWithIssues, Failed, Canceled, Skipped
Write-Host "##vso[task.complete result=SucceededWithIssues;]Task finished with minor linting warnings."
```

---

## 4. Identidad del Pipeline: `build.updatebuildnumber` y `build.addbuildtag`

Modifican los metadatos de la ejecución para que sea fácil de identificar en el historial.

### Casos de uso
```powershell
# Change the execution name (e.g., from '20231010.1' to a semantic version)
Write-Host "##vso[build.updatebuildnumber]2.1.0-beta+$(Build.BuildId)"

# Add tags to filter or categorize the run
Write-Host "##vso[build.addbuildtag]Production"
Write-Host "##vso[build.addbuildtag]SecurityAudited"
```

---

## 5. Reportes y Evidencias: `task.upload*` y `task.addattachment`

Permiten adjuntar archivos y crear reportes personalizados.

### Casos de uso
```powershell
# Upload a specific file associated with this task's logs
Write-Host "##vso[task.uploadfile]C:\logs\memory_dump.log"

# Create a custom Markdown tab in the Pipeline Summary (Golden Command!)
Write-Host "##vso[task.uploadsummary]$(System.DefaultWorkingDirectory)/reports/quality_check.md"

# Attach data for third-party extensions or specialized tools
Write-Host "##vso[task.addattachment type=DistributedTest;name=TestResults;]C:\tests\results.xml"
```

---

## 6. Conexiones y Telemetría: `task.setendpoint` y `telemetry.publish`

Para automatización avanzada y recolección de métricas.

### Casos de uso
```powershell
# Update a field in a Service Connection dynamically (Use with caution)
Write-Host "##vso[task.setendpoint id=ENDPOINT_ID;field=authParameter;key=password]NewSecretPassword"

# Send custom telemetry data to Azure DevOps for performance analysis
Write-Host "##vso[telemetry.publish area=DeploymentStats;feature=Duration] { 'app': 'API', 'time_sec': 345 }"
```

### Conexiones: `task.setendpoint`

Este comando te permite **modificar los datos de esa conexión mientras el pipeline está corriendo**. 

Normalmente, una Service Connection es estática: tiene una URL y una clave fijas. Pero con `task.setendpoint`, tu script puede decirle al Agente: *"Oye, para esta conexión específica, cambia la URL o actualiza este Token antes de que la siguiente tarea lo use"*.

```powershell
Write-Host "##vso[task.setendpoint id=ENDPOINT_ID;field=PROPERTY;key=KEY_NAME]NEW_VALUE"
```

Los parámetros clave:
* **`id`**: Es el UUID (el identificador largo de sistema) de la Service Connection. No es el nombre que le pusiste, sino el ID que ves en la URL cuando la editas en el navegador.
* **`field`**: El área que quieres tocar. Los más comunes son `authParameter` (para credenciales) o `url`.
* **`key`**: El nombre del campo específico dentro de ese área (ej: `username`, `password`, `apitoken`).
* **`NEW_VALUE`**: El nuevo valor que quieres inyectar.

---

## Tabla de Referencia Rápida

| Comando | Acción Principal | Impacto Visual | Nivel |
| :--- | :--- | :--- | :--- |
| `task.setvariable` | Persistir datos | Ninguno (interno) | Básico |
| `task.debug` | Log de diagnóstico | Azul (si activo) | Básico |
| `task.issue` | Alertas | Amarillo / Rojo | Básico |
| `task.setprogress` | Mostrar % de carga | Barra de progreso | Medio |
| `build.updatebuildnumber` | Renombrar ejecución | Título del Build | Medio |
| `task.uploadsummary` | Reporte Markdown | Pestaña "Summary" | Medio |
| `task.complete` | Forzar fin de tarea | Estado del Step | Avanzado |
| `task.setendpoint` | Modificar conexiones | Ninguno (interno) | Experto |

---

## Reglas de Oro para el uso de `##vso`

1.  **Canal de salida:** Usa siempre `Write-Host` en PowerShell (o `echo` en Bash). El Agente escucha la consola (stdout). Si usas `Write-Output`, el comando podría ser capturado por una variable y nunca llegar al Agente.
2.  **Sintaxis estricta:** El formato es `##vso[AREA.COMANDO PROPIEDAD=VALOR;]MENSAJE`. Los corchetes y el punto y coma son obligatorios.
3.  **Seguridad:** Si generas un secreto, usa `issecret=true` inmediatamente. Azure DevOps enmascarará ese valor retroactivamente en los logs de esa ejecución.
4.  **Ubicación:** El comando debe estar al inicio de una línea nueva para que el Agente lo procese correctamente.
