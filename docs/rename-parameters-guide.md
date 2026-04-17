# Guía: Renombrado parent* → conditional*

Este documento explica los pasos y comandos para repetir de forma segura el renombrado:
- `parentStageName` → `conditionalStage`
- `parentSummaryJobName` → `conditionalJob`
- `parentSummaryTaskName` → `conditionalTask`
- `parentSummaryVariableName` → `conditionalVariable`

**Resumen rápido**
- Crea una rama, aplica cambios en plantillas, actualiza callers, revisa docs, valida con linter y genera un parche.

**Pasos recomendados**
- **Branch**: `git checkout -b rename-parent-to-conditional`
- **Editar plantillas**: actualizar las `parameters` y todas las referencias internas en:
  - azure-devops-pipelines/scripts-templates/jobs/conditioned-job-wrapper.yml
  - azure-devops-pipelines/scripts-templates/jobs/conditioned-job.yml
  - azure-devops-pipelines/scripts-templates/stages/conditioned-stage.yml
- **Actualizar callers**: buscar con `git grep` y actualizar los mappings en los YAML que pasan esos parámetros (root `azure-pipelines.yml`, templates consumidores, etc.).

**Comandos útiles**
- Buscar ocurrencias:

```
git grep -n "parentStageName\|parentSummaryJobName\|parentSummaryTaskName\|parentSummaryVariableName" || true
```

- Reemplazo rápido (PowerShell — PRUEBA en rama, luego revisa diffs):

```powershell
Get-ChildItem -Recurse -Include *.yml,*.yaml,*.md |
  ForEach-Object {
    (Get-Content $_.FullName) -replace '\bparentStageName\b','conditionalStage' -replace '\bparentSummaryJobName\b','conditionalJob' -replace '\bparentSummaryTaskName\b','conditionalTask' -replace '\bparentSummaryVariableName\b','conditionalVariable' | Set-Content $_.FullName
  }
```

- Alternativa (WSL/sed):

```bash
grep -RIl --exclude-dir=.git -e 'parentStageName' -e 'parentSummaryJobName' -e 'parentSummaryTaskName' -e 'parentSummaryVariableName' | xargs sed -E -i 's/\bparentStageName\b/conditionalStage/g; s/\bparentSummaryJobName\b/conditionalJob/g; s/\bparentSummaryTaskName\b/conditionalTask/g; s/\bparentSummaryVariableName\b/conditionalVariable/g'
```

**Precauciones**
- No ejecutar un reemplazo global en `.md` sin revisión: actualiza solo bloques de código y ejemplos.
- Si las plantillas se consumen desde repos remotos, coordina la actualización cross-repo.
- No renombres valores literales que representan nombres exportados en runtime (p. ej. el *valor* de una variable). Aquí renombramos nombres de parámetros y sus referencias.

**Validación**
- Buscar restos: `git grep -n "parentStageName\|parentSummaryJobName\|parentSummaryTaskName\|parentSummaryVariableName"`
- Lint YAML: instalar y ejecutar `yamllint` sobre los templates modificados:

```bash
pip install yamllint
yamllint azure-devops-pipelines/scripts-templates/*.yml
```

- Comprobar que los pipelines consumidores (p. ej. los `dummy-*`) se expanden/compilan en Azure DevOps (usar UI o `az pipelines` si está configurado).

**Generar parche**
- Después de ajustar y revisar `git diff`:

```bash
# desde la raíz del repo
git diff > rename-parent-to-conditional.patch   # guarda cambios locales no commit
# o si ya commiteaste
git format-patch -1 HEAD --stdout > rename-parent-to-conditional.patch
# comprobar el parche
git apply --check rename-parent-to-conditional.patch
```

**Script seguro (solo para uso como referencia)**
- El siguiente script en Python reemplaza en archivos `.yml`/`.yaml` directamente y en bloques de código YAML dentro de `.md` (modo `--check` para simulación, `--apply` para aplicar):

```python
#!/usr/bin/env python3
"""
rename_parent_to_conditional.py
Usage:
  python rename_parent_to_conditional.py --check   # dry run
  python rename_parent_to_conditional.py --apply   # apply changes
"""
import re
from pathlib import Path
import argparse

MAPPING = {
  'parentStageName': 'conditionalStage',
  'parentSummaryJobName': 'conditionalJob',
  'parentSummaryTaskName': 'conditionalTask',
  'parentSummaryVariableName': 'conditionalVariable',
}
WORD_RE = re.compile(r"\\b(" + "|".join(re.escape(k) for k in MAPPING) + r")\\b")

def replace_text(s):
    return WORD_RE.sub(lambda m: MAPPING[m.group(1)], s)

def process_yaml(path, apply):
    s = path.read_text(encoding='utf8')
    ns = replace_text(s)
    if ns != s:
        print('CHANGE:', path)
        if apply:
            path.write_text(ns, encoding='utf8')
        return True
    return False

def process_md(path, apply):
    s = path.read_text(encoding='utf8')
    out = []
    inside = False
    lang = ''
    changed = False
    for line in s.splitlines(True):
        if not inside:
            m = re.match(r'^```(\\w+)?', line)
            if m:
                inside = True
                lang = (m.group(1) or '').lower()
                out.append(line)
                continue
            out.append(line)
        else:
            if line.startswith('```'):
                inside = False
                lang = ''
                out.append(line)
                continue
            if lang in ('yaml','yml'):
                nl = replace_text(line)
                if nl != line:
                    changed = True
                out.append(nl)
            else:
                out.append(line)
    ns = ''.join(out)
    if ns != s:
        print('CHANGE md:', path)
        if apply:
            path.write_text(ns, encoding='utf8')
        return True
    return False

if __name__ == '__main__':
    p = argparse.ArgumentParser()
    p.add_argument('--apply', action='store_true')
    p.add_argument('--root', default='.')
    args = p.parse_args()
    root = Path(args.root)
    files = list(root.rglob('*.yml')) + list(root.rglob('*.yaml')) + list(root.rglob('*.md'))
    any_change = False
    for f in files:
        if f.suffix.lower() in ('.yml', '.yaml'):
            changed = process_yaml(f, args.apply)
        else:
            changed = process_md(f, args.apply)
        any_change = any_change or changed
    print('Dry run changes found' if not args.apply else 'Applied changes', any_change)
```

**Notas finales**
- Revisa siempre el diff antes de commitear.
- Si vas a publicar la plantilla a otras repos, incrementa la versión/tag para evitar romper consumidores.

---
Guarda este fichero y úsalo como referencia. Si quieres, genero también un script standalone en `scripts/` y la `rename-parent-to-conditional.patch` con los cambios ya realizados.