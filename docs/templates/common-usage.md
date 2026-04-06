# Common Usage

## Minimal ASP.NET

```yaml
resources:
  repositories:
    - repository: azure-devops-pipelines
      type: git
      name: https://github.com/hfrances/azure-devops-pipelines
      ref: refs/tags/3.3.0

extends:
  template: standard/azure-pipelines-template-aspnet.yml@azure-devops-pipelines
  parameters:
    Deploy: auto
    Analyze: disabled
    Test: auto
    Solution: '*.sln'
    SupportsUniversalPackages: false
    SupportsContainers: true
```

## Minimal Library

```yaml
resources:
  repositories:
    - repository: azure-devops-pipelines
      type: git
      name: https://github.com/hfrances/azure-devops-pipelines
      ref: refs/tags/3.3.0

extends:
  template: standard/azure-pipelines-template-libnet.yml@azure-devops-pipelines
  parameters:
    Deploy: auto
    Analyze: auto
    Test: auto
    Solution: '*.sln'
    SupportsNuget: true
```

## Deploy Modes

### `Deploy: auto`

Use branch rules from the template.

### `Deploy: true`

Keep publish eligible on every branch, then let change detection decide.

### `Deploy: false`

Disable the main publish path controlled by `PublishEnabled`.

## Change Detection

Enable:

```yaml
PublishChangeDetection: true
```

Disable:

```yaml
PublishChangeDetection: false
```

When disabled:
- build goes back to always-build behavior
- publish goes back to the parent template intent

## ASP.NET Extra Publish File

```text
.azuredevops/publish-extra-paths.txt
```

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
