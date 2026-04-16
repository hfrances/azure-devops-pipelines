# Component Catalog Variable Patterns

## Objetivo
Definir un patron comun para declarar `variables:` en pipelines con catalogo de componentes (`single` y `combi`) y evitar drift entre repos.

## Reglas base
- `PublishChangeDetection` debe ser el primer parametro runtime.
- `PublishChangeDetection.displayName` debe ser exactamente: `Enable change detection`.
- Usar comentarios por grupos funcionales dentro de `variables:`.
- Usar `private/public` para clasificar contenedores por exposicion.
- Usar `external/internal` solo para `acaIngressType`.
- No declarar `dependsOn: []` cuando no hay dependencias.

## Orden recomendado en `variables:`
1. `# Docker`
2. `# Platform`
3. `# Container (private)`
4. `# Container (public)` (si aplica)
5. `# Container (shared characteristics)` (si aplica)

## Detalle por grupo

### `# Docker`
Variables de build/publicacion de imagen:
- `dockerRegistryServiceConnection`
- `imageRepository` o repositorios por componente (`app1ImageRepository`, etc.)

### `# Platform`
Variables transversales de Azure:
- `azureServiceConnection`
- `resourceGroup`
- `resourceGroupVNet`
- `acrName`
- `identityName`
- `vnetName`
- `subnetName`
- `caeLocation`

### `# Container (private)`
Identidades del despliegue privado (normalmente en VNet o CAE privado):
- `caeName`, `acaName`
- o equivalentes por componente (`backAppName`, `frontAppName`, etc.)

### `# Container (public)`
Solo si hay despliegue publico adicional:
- `caeNamePublic`, `acaNamePublic`
- o equivalentes por componente

### `# Container (shared characteristics)`
Dimensionado comun reusable entre private/public:
- `acaTargetPort` o `appTargetPort`
- `acaCpu` / `appCpu`
- `acaMemory` / `appMemory`
- `acaMinReplicas` / `appMinReplicas`
- `acaMaxReplicas` / `appMaxReplicas`

## Patrones por tipo de pipeline

### Single
- Un unico componente en `components[]`.
- Puede tener uno o dos deploy jobs (private/public) reutilizando la misma imagen.
- En `components[]`, el campo `component` debe quedar vacio.
- En jobs de deploy, no declarar `component.name`.

Ejemplo (single):
```yaml
components:
  - component:
    templateType: pnpm
    workingDirectory: .
```

Ejemplo (deploy single):
```yaml
component:
  azureServiceConnection: $(azureServiceConnection)
  resourceGroup: $(resourceGroup)
```

### Combi
- Multiples componentes en `components[]`.
- Dependencias por componente con `component.dependsOn`.
- Variables de contenedor separadas por componente cuando cada uno tiene `acaName`/puerto propios.

## Reutilizacion private/public
Si private y public despliegan la misma app:
- Declarar primero variables compartidas (imagen, cpu/memoria, replicas, puertos).
- Declarar despues solo las diferencias (por ejemplo `caeName/acaName` de public).

Esto reduce duplicacion y hace mas facil mantener ambos despliegues sincronizados.
