# Library Template

Template: `standard/azure-pipelines-template-libnet.yml`

## Use It For

- reusable .NET libraries
- repos that pack and push NuGet packages

## Main Params

| Param | Default | What it does |
| --- | --- | --- |
| `Deploy` | `false` | Controls NuGet publish intent |
| `Analyze` | `auto` | SonarCloud |
| `Test` | `auto` | Tests + coverage |
| `Solution` | `*.sln` | Solution pattern |
| `ForceVersion` | `true` | Version replacement |
| `SupportsNuget` | `true` | NuGet pack/push steps |
| `PublishChangeDetection` | `true` | Skip work when nothing publishable changed |

Internally, `prepare-dotnet.yml` also supports `jobName`, `displayName`, `tagSuffix`, `publishEnabled` and `publishDocker`. The standard library template keeps the default single-artifact behavior unless a consumer overrides them explicitly.

## You Need

| Variable | Needed for |
| --- | --- |
| `ExternalFeed` | external restore if used |
| `FileTransformEnabled` | JSON transform |
| `SonarCloudOrganization` | SonarCloud in repos that use it |

## Publish Rules

`PublishEnabled`:
- `Deploy=true` -> publish allowed on any branch
- `Deploy=auto` -> publish allowed on `master`, `main`, `staging`, `alpha`, `feature/net10`
- `Deploy=false` -> publish disabled

Then change detection can still switch publish off if nothing publishable changed through:
- `HasAffectedPublishProjects`
- `PublishEnabled`

| Branch | `auto` | `true` | `false` |
| --- | --- | --- | --- |
| `master` | maybe publish | maybe publish | no publish |
| `main` | maybe publish | maybe publish | no publish |
| `staging` | maybe publish | maybe publish | no publish |
| `alpha` | maybe publish | maybe publish | no publish |
| `development` | no publish | maybe publish | no publish |

`PublishDocker` is always `false` here.

## Pipeline Flow

1. `Prepare`: checkout, resolve solution/project, read project properties, run change detection
2. `Build`: map `Prepare` outputs into `BuildParameters.*`
3. Restore
4. Run cancellation audit
5. Build/test/analyze if `BuildParameters.HasAffectedBuildProjects=true`
6. Tag `compiled`
7. Pack if `BuildParameters.HasAffectedPublishProjects=true`
8. Push to NuGet if `BuildParameters.HasAffectedPublishProjects=true` and `PublishEnabled=true`
9. Tag `published`

## Important Detail

Pack and push are different:
- `NuGet: Pack` depends on `BuildParameters.HasAffectedPublishProjects`
- `NuGet: Push` depends on both `BuildParameters.HasAffectedPublishProjects` and `PublishEnabled`

So you can still get a package artifact with `Deploy=false`.
