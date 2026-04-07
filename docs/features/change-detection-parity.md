# Change Detection Parity Guide (`dotnet` vs `git`)

## Scope

This document defines how to maintain parity between:

- `scripts-templates/detect-publishable-changes-dotnet.yml`
- `scripts-templates/detect-publishable-changes-git.yml`

Both templates solve the same problem (build/publish change detection), with one intentional functional difference:

- `.NET` uses `dotnet-affected` for semantic project-graph analysis.
- `git` uses path-based analysis only.

## Canonical Template

For structure and formatting parity, the canonical reference is:

- `detect-publishable-changes-dotnet.yml`

The `git` template should mimic it as closely as possible in:

- ordering of the main flow
- line structure of core control blocks
- compact one-line style where already used
- semicolon style and statement grouping

## Parity Concepts

### 1. Behavioral parity

Shared behavior must stay aligned:

- same baseline lookup strategy (`compiled` / `published`, optional suffix)
- same fail-open policy (`build=true`, `publish=true` on detector failure)
- same output variable names:
  - `BuildParameters.HasAffectedBuildProjects`
  - `BuildParameters.HasAffectedPublishProjects`
- same `enabled` behavior:
  - `enabled=true` => full detection
  - `enabled=false` => keep default affected flags (`true/true`)

### 2. Structural parity

Core flow should keep the same block order:

1. parameter + local init
2. baseline section
3. `try { ... } catch { ... }`
4. changed-files reporting
5. export output variables
6. final decision logs

### 3. Style parity ("obfuscation" / compact style)

These templates intentionally use compact inline PowerShell in critical sections.
When changing one template, keep equivalent lines in the other template with the same compact style where behavior is shared.

Do not reformat only one template unless the same style change is applied (or intentionally documented) in the other.

## Organization Rules

In `dotnet`, place functions in this order:

1. Common functions shared conceptually with `git`
2. Dotnet-exclusive functions grouped together

Recommended grouping:

- Common:
  - `log`, `markissue`, `blank`, `section`, `bool`, `norm`, `inwd`, `relwd`, `any`
  - `baseline`, `gitok`, `gitlines`
  - `hasbuildchanges`, `haspublishinputchanges`
  - `cfg`, `suffixname`, `report`
- Dotnet-exclusive:
  - `underwd`, `publishable`, `affected`, `copydrop`, `ensuretool`

## Why helper methods were introduced

To keep high-line parity in core flow, some decision points are wrapped in helpers:

- `hasbuildchanges(...)`
- `haspublishinputchanges(...)`

This keeps the main `if($cb){...}` and `if($pb){...}` lines aligned across templates while allowing intentional behavior differences inside helper implementations.

## Difference Table

| Aspect | `dotnet` detector | `git` detector |
|---|---|---|
| Initial change source | `git diff` | `git diff` |
| Build decision | `git diff` + `dotnet-affected` refinement | `git diff` + include/exclude path filters |
| Publish pre-check | commit/baseline + input-change check | commit/baseline + input-change check |
| Publish final decision | affected publishable projects + optional ASP.NET extras | path-based include/exclude only |
| Project-level publish filter | `publishable(...)` excludes tests/examples | N/A (no project graph) |
| Include/exclude usage | mainly publish extras (`aspnet` mode) | build + publish path gating |
| Dotnet-specific step | `dotnet-affected` execution and outputs | none |

## Safe Change Protocol

When editing either detector:

1. Apply behavioral change first in canonical (`dotnet`) if shared logic changes.
2. Mirror equivalent structural/style change to `git` immediately.
3. Keep non-shared behavior isolated in helper functions.
4. Re-run a side-by-side diff and confirm:
   - control flow blocks remain aligned
   - only intentional differences remain
5. Update `docs/features/change-detection.md` if behavior changed.

## Intentional Differences (Allowed)

Differences are acceptable only when they are:

- semantically required (`dotnet-affected`, `publishable`)
- documented in this file
- isolated to helper implementations or dotnet-exclusive blocks

Everything else should be treated as drift and aligned back to parity.
