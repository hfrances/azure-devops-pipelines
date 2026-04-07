# Change Detection

Templates:
- `.NET`: `jobs-templates/prepare-dotnet.yml` + `scripts-templates/detect-publishable-changes-dotnet.yml`
- `git`: `jobs-templates/prepare-path-change-git.yml` + `scripts-templates/detect-publishable-changes-git.yml`

## What It Does

It decides:
- should this run build?
- should this run publish?

It compares the current commit with the latest successful run tagged:
- `compiled`
- `published`

If `tagSuffix` is set, it compares against:
- `compiled-<suffix>`
- `published-<suffix>`

## .NET Inputs

| Param | Default | Meaning |
| --- | --- | --- |
| `enabled` | `true` | enables/disables detector logic; when `false`, both affected flags are set to `true` |
| `workingDirectory` | `''` | optional base folder used to scope git changes and resolve relative paths |
| `solutionPath` | `$(Solution.FullName)` | solution for `dotnet-affected` |
| `tagSuffix` | `''` | optional suffix for tags like `compiled-api` / `published-api` |
| `publishExtraMode` | `none` | `aspnet` enables extra non-project publish files |
| `includedPatterns` | `[]` | optional wildcard patterns that replace the default ASP.NET extra include list |
| `excludedPatterns` | `[]` | optional wildcard patterns that replace the default ASP.NET extra exclude list |

## .NET Outputs

The detection step publishes these outputs:

- `HasAffectedBuildProjects`
- `HasAffectedPublishProjects`

In the standard .NET templates they are produced in the `Prepare` job and then mapped into the `Build` job variables before any gated task runs.

`prepare-dotnet.yml` also supports:
- `jobName`
- `displayName`
- `workingDirectory`
- `includedPatterns`
- `excludedPatterns`
- `tagSuffix`

When `workingDirectory` is set:
- git diff is scoped to that subtree before build/publish decisions are made
- `solutionPattern` in `prepare-dotnet.yml` is evaluated from that directory, so it must be relative to `workingDirectory`
- ASP.NET extra publish path rules are evaluated relative to `workingDirectory`

## .NET Build Decision

Build becomes `false` when:
- current commit already matches last `compiled` run
- there are no changed files since that baseline
- `dotnet-affected` finds no affected build projects

## .NET Publish Decision

Publish becomes `false` when:
- current commit already matches last `published` run
- there are no changed files since that baseline
- `dotnet-affected` finds no publishable projects
- in `aspnet` mode, no extra publish files match

## .NET Important Distinction

- `HasAffectedBuildProjects`: do we need build/test/analyze work?
- `HasAffectedPublishProjects`: do we need publish/package work?

These are related, but not the same.

## .NET ASP.NET Extra Files

Default extra publish matches:

```text
**/Dockerfile
**/*.json
**/*.xml
**/*.config
**/*.resx
**/wwwroot/**
```

If `includedPatterns` is informed, it replaces that default include list.
If `excludedPatterns` is informed, it replaces the default exclude list.

Override file in the consumer repo:

```text
.azuredevops/publish-extra-paths.txt
```

If `workingDirectory` is set, the detector first looks for:

```text
<workingDirectory>/.azuredevops/publish-extra-paths.txt
```

and falls back to the repo-root file only if that scoped file does not exist.

Example:

```text
[include]
**/Dockerfile
**/*.json
**/wwwroot/**

[exclude]
README*
docs/**
**/bin/**
**/obj/**
```

## .NET Drop Files

Always:
- `changed-files-build.txt`
- `changed-files-deploy.txt`

Sometimes:
- `affected-build.proj`
- `affected-publish.proj`

If `tagSuffix` is set, the exported file names also use that suffix. Example:
- `changed-files-build-api.txt`
- `changed-files-deploy-api.txt`
- `affected-build-api.proj`
- `affected-publish-api.proj`

## .NET Failure Mode

If detection breaks, it fails open:
- build stays on
- publish stays on

## Git Inputs

| Param | Default | Meaning |
| --- | --- | --- |
| `enabled` | `true` | enables/disables detector logic; when `false`, both affected flags are set to `true` |
| `workingDirectory` | `''` | optional subtree used to scope git changes |
| `tagSuffix` | `''` | optional suffix for tags like `compiled-ui` / `published-ui` |
| `includedPatterns` | `[]` | optional wildcard patterns that replace the implicit include default |
| `excludedPatterns` | `[]` | optional wildcard patterns that replace the default exclude list |

`prepare-path-change-git.yml` also supports:
- `jobName`
- `displayName`
- `publishChangeDetection`
- `workingDirectory`
- `tagSuffix`
- `includedPatterns`
- `excludedPatterns`

## Git Behavior

The `git` detector is intended for components without a .NET solution graph, such as Next.js, React, Vue, or any path-scoped artifact inside a monorepo.

When `workingDirectory` is set:
- git diff is restricted to that subtree
- include/exclude matching is evaluated relative to that subtree
- `.azuredevops/publish-extra-paths.txt` is resolved from `<workingDirectory>` first, then from repo root

Build and publish decisions are path-based:
- `build` compares against the latest successful `compiled[-suffix]` baseline
- `publish` compares against the latest successful `published[-suffix]` baseline
- both become `false` when the current commit already matches the baseline, when there are no changed files in scope, or when no changed files survive include/exclude filtering

Default `git` include/exclude rules:

```text
include:
**

exclude:
README*
**/README*
docs/**
.github/**
.azuredevops/**
azure-pipelines*.yml
**/azure-pipelines*.yml
```

The same override file format is supported:

```text
.azuredevops/publish-extra-paths.txt
```

Artifacts exported by the `git` detector:
- `changed-files-build.txt`
- `changed-files-deploy.txt`

If `tagSuffix` is set, the exported file names include it. Example:
- `changed-files-build-ui.txt`
- `changed-files-deploy-ui.txt`

Failure mode is also fail-open:
- build stays on
- publish stays on
