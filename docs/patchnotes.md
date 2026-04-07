# Patch Notes

## 3.5.0

Status: current working tree, pending tag.

Highlights:
- add shared `aca-deploy-job` documentation under `docs/features`
- update `aca-deploy-job` to print the hosts-file warning only for external ingress on subnet-based environments
- document the external-ingress-only hosts warning behavior
- rename `calculate-release-version-dotnet.yml` to `calculate-release-version.yml` with optional `calculateAssemblyVersion` parameter
- Add Node.js templates `apply-release-version-node.yml` and `get-project-properties-node.yml`

Main files:
- `jobs-templates/aca-deploy-job.yml`
- `docs/features/aca-deploy-job.md`
- `scripts-templates/calculate-release-version.yml` (renamed from `calculate-release-version-dotnet.yml`)
- `scripts-templates/apply-release-version-node.yml` (moved from dummy-common)
- `scripts-templates/get-project-properties-node.yml` (moved from dummy-common)

## 3.4.0

Status: previous working tree release notes.

Highlights:
- add publish change detection with baseline comparison and `dotnet-affected`
- add `mark-build-published.yml` and tag successful runs as `compiled` / `published`
- update `aspnet` and `libnet` templates to skip build or publish work when nothing relevant changed
- add conditional support to shared script templates
- expand docs with template guides, feature guides, and usage examples
- keep ASP.NET extra publish file support through `.azuredevops/publish-extra-paths.txt`

Main files:
- `scripts-templates/detect-publishable-changes-dotnet.yml`
- `scripts-templates/mark-build-published.yml`

## 3.3.0

Tag: `3.3.0`  
Date: `2026-03-11`  
Commit: `dcbd5a1ae16541d7c975884872b6b9003bc0721f`

Highlights:
- improve `aspnet` and `libnet` standard templates
- add the cancellation audit template
- improve target framework calculation
- improve main project detection
- update release version and shared helper templates

## 3.2.0

Tag: `3.2.0`  
Date: `2026-02-26`  
Commit: `7bb701ee92d2a8775fbf55ec78529f431fcb9af9`

Highlights:
- update the `libnet` template only
- publish test results directly from the `dotnet test` step
- widen test project matching from only `*.csproj` to `*.??proj`
- add `TestEnabled` and pass test-related values into target framework calculation
- add support for `10.0` in `SupportedFrameworks`
- use pipeline variables for SonarCloud organization and external feed credentials
- add restore inputs for `feedsToUse`, `nugetConfigPath`, and external feed credentials
- gate SonarCloud and NuGet publish steps more explicitly
- update coverage helper usage, including `reportgenerator@5`

Notes:
- this change only affected `standard/azure-pipelines-template-libnet.yml`
- `aspnet` stayed on `3.1.0` at this point

## 3.1.0

Tag: `3.1.0`  
Date: `2025-07-28`  
Commit: `d18d80defe856e20994211630122bba87a8419ce`

Highlights:
- initial implementation of `aspnet` and `libnet` templates
- initial shared script templates for versioning, target frameworks, test detection, coverage, and project discovery

Main files introduced:
- `standard/azure-pipelines-template-aspnet.yml`
- `standard/azure-pipelines-template-libnet.yml`
- `scripts-templates/calculate-release-version-dotnet.yml`
- `scripts-templates/calculate-target-frameworks-dotnet.yml`
- `scripts-templates/get-main-project-dotnet.yml`
