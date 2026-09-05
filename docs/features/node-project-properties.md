# Node Project Properties

Template: `scripts-templates/get-project-properties-node.yml`

## What It Does

Reads the main Node package manifest, validates that its `Dockerfile` exists,
prints the detected properties, and exports them for downstream tasks.

## Inputs

| Param | Default | Meaning |
| --- | --- | --- |
| `displayName` | `Get main Node project properties` | task display name |
| `workingDirectory` | `.` | component directory containing `package.json` and `Dockerfile` |

## Output Variables

- `MainProject.FullName`
- `MainProject.Version`
- `MainProject.PackageName`
- `MainProject.Description`
- `NodeProject.HasTestScript`
- `NodeProject.PackageManager`
- `NodeProject.DockerfilePath`
- `NodeProject.WorkingDirectory`

## Resolution Rules

- `packageManager` is read from `package.json` when present.
- Otherwise, the package manager is inferred from `pnpm-lock.yaml`,
  `package-lock.json`, or `yarn.lock`.
- When no version is provided, the helper uses `0.1.0` and logs a warning.

The task fails when `package.json` or `Dockerfile` is missing.
