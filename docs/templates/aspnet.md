# ASP.NET Template

Template: `standard/azure-pipelines-template-aspnet.yml`

## Use It For

- ASP.NET apps
- APIs
- services
- projects that may publish Universal Packages or Docker images

## Main Params

| Param | Default | What it does |
| --- | --- | --- |
| `Deploy` | `false` | Controls Universal Package publish intent |
| `Analyze` | `disabled` | SonarCloud |
| `Test` | `auto` | Tests + coverage |
| `Solution` | `*.sln` | Solution pattern |
| `ForceVersion` | `false` | Version replacement |
| `SupportsUniversalPackages` | `false` | Universal Package publish steps |
| `SupportsContainers` | `true` | Docker steps |
| `PublishChangeDetection` | `true` | Skip work when nothing publishable changed |

Internally, `prepare-dotnet.yml` also supports `jobName`, `displayName`, `tagSuffix`, `publishEnabled` and `publishDocker`. The standard ASP.NET template keeps the default single-artifact behavior unless a consumer overrides them explicitly.

## You Need

| Variable | Needed for |
| --- | --- |
| `ContainerRegistry` | Docker push |
| `ContainerName` | Docker push |
| `ExternalFeed` | external restore if used |
| `FileTransformEnabled` | JSON transform |

## Publish Rules

### Universal Packages

`PublishEnabled`:
- `Deploy=true` -> publish allowed on any branch
- `Deploy=auto` -> publish allowed only on `master` or `main`
- `Deploy=false` -> publish disabled

Then change detection can still switch publish off if nothing publishable changed through:
- `HasAffectedPublishProjects`
- `PublishEnabled`

| Branch | `auto` | `true` | `false` |
| --- | --- | --- | --- |
| `master` | maybe publish | maybe publish | no publish |
| `main` | maybe publish | maybe publish | no publish |
| `staging` | no publish | maybe publish | no publish |
| `development` | no publish | maybe publish | no publish |

### Docker

Current implementation:
- `PublishDocker` starts as `true`
- it is not branch-gated by `Deploy=auto`
- change detection can switch it off through `PublishDocker`

So today Docker is more permissive than Universal Packages.

| Branch | `auto` | `true` | `false` |
| --- | --- | --- | --- |
| `master` | maybe publish | maybe publish | maybe publish |
| `main` | maybe publish | maybe publish | maybe publish |
| `staging` | maybe publish | maybe publish | maybe publish |
| `development` | maybe publish | maybe publish | maybe publish |

`maybe publish` means:
- `HasAffectedPublishProjects=true`
- change detection publish intent says yes
- `SupportsContainers=true`
- `ContainerRegistry` and `ContainerName` are set

## Pipeline Flow

1. `Prepare`: checkout, resolve solution/project, read project properties, run change detection
2. `Build`: map `Prepare` outputs into `BuildParameters.*`
3. Restore
4. Run cancellation audit
5. Build/test/analyze if `BuildParameters.HasAffectedBuildProjects=true`
6. Tag `compiled`
7. Publish output if `BuildParameters.HasAffectedPublishProjects=true`
8. Publish Universal Package and/or Docker only if `BuildParameters.HasAffectedPublishProjects=true`
9. Tag `published`

## What Commonly Gets Skipped

Build side:
- current commit already compiled
- no changed files since compiled baseline
- no affected build projects

Publish side:
- current commit already published
- no changed files since published baseline
- no publishable projects
- no matching ASP.NET extra publish files

## Extra Publish Files

ASP.NET change detection can republish because of files like:
- `Dockerfile`
- `*.json`
- `*.xml`
- `*.config`
- `*.resx`
- `wwwroot/**`

You can override the matching rules with:

```text
.azuredevops/publish-extra-paths.txt
```

Full format: [Change detection](../features/change-detection.md)
