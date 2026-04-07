# Release Version

Template: `scripts-templates/calculate-release-version.yml`

## What It Does

Turns `$(MainProject.Version)` into pipeline variables used later for:
- package version
- assembly version (optional)
- Docker version
- Docker alias

It also adds build tags with those values.

## Inputs

| Param | Default | Meaning |
| --- | --- | --- |
| `ProjectVersion` | `$(MainProject.Version)` | Source version read from the main project |
| `condition` | `succeeded()` | Azure Pipelines condition |
| `calculateAssemblyVersion` | `true` | Whether to calculate and set `ReleaseAssemblyVersion` (useful for .NET projects) |

## Output Variables

- `BuildParameters.ReleaseVersion`
- `BuildParameters.ReleaseAssemblyVersion` (if `calculateAssemblyVersion` is `true`)
- `BuildParameters.DockerVersion`
- `BuildParameters.DockerAlias`

It also adds build tags:
- assembly version (if `calculateAssemblyVersion` is `true`)
- Docker version
- Docker alias

## Rules

### If the project version already has a prerelease

Example:

```text
1.2.3-beta
1.2.3-beta.1
```

Result:
- `ReleaseVersion = <ProjectVersion>+<Build.BuildId>`
- `DockerVersion = <baseVersion>-<cleanPrerelease>`
- `DockerAlias = first word-like part of prerelease`

Example:

```text
ProjectVersion = 1.2.3-beta.1
Build.BuildId = 123

ReleaseVersion = 1.2.3-beta.1+123
DockerVersion = 1.2.3-beta1
DockerAlias = beta
```

### If there is no prerelease

| Branch | ReleaseVersion | DockerVersion | DockerAlias |
| --- | --- | --- | --- |
| `master` / `main` | `<version>` | `<version>` | `latest` |
| `staging` | `<version>-alpha.<buildId>` | `<version>-alpha.<buildId>` | `alpha` |
| any other branch | `<version>-<cleanBranch>.<buildId>` | `<version>-<cleanBranch>.<buildId>` | `<cleanBranch>` |

## Assembly Version

When `calculateAssemblyVersion` is `true` (default), always:

```text
<major>.<minor>.<patch>.<Build.BuildId>
```

If the source version is missing `patch`, the template treats it as `0`.

## Sanitizing

The template removes non-alphanumeric characters when building:
- prerelease Docker suffixes
- branch-based Docker aliases

Examples:
- `feature/my-api` -> `featuremyapi`
- `beta.1` -> `beta1`

## Failure Mode

If `ProjectVersion` does not match the expected version regex, the task fails with:

```text
Invalid version format for: <ProjectVersion>
```
