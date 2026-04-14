# 📦 Azure-Devops-Pipelines (Reusable Pipeline Templates)

This repository contains a collection of reusable **Azure Pipelines YAML templates** designed to standardize and simplify CI/CD workflows across multiple projects.

## 🧩 Index

- Templates:
  - [Common Usage](docs/templates/common-usage.md)
  - [ASP.NET Template](docs/templates/aspnet.md)
  - [Library Template](docs/templates/libnet.md)
- Features:
  - [Change Detection](docs/features/change-detection.md)
  - [Release Version](docs/features/release-version.md)
  - [Cancellation Audit](docs/features/cancellation-audit.md)
  - [ACA Deploy Job](docs/features/aca-deploy-job.md)
- Releases:
  - [Patch Notes](docs/patchnotes.md)

## 🚀 How to Use

Reference this repository from the consumer pipeline:

```yaml
resources:
  repositories:
    - repository: hf-azure-pipelines-repo
      type: git
      name: https://github.com/hfrances/azure-devops-pipelines
      ref: refs/heads/master  # or a specific tag/branch

extends:
  template: standard/azure-pipelines-template-aspnet.yml@hf-azure-pipelines-repo
  parameters:
    param1: value1
    param2: value2
```

Canonical alias policy:
- Use `hf-azure-pipelines-repo` as the repository alias for this template repository.
- Do not use alternative aliases if you want nested template calls to resolve consistently.

For template behavior, branch rules, required variables, and optional files such as `.azuredevops/publish-extra-paths.txt`, use the docs linked above.

## 📖 Official Documentation

For more details on how to structure and use templates in Azure Pipelines, refer to the official Microsoft documentation:  
👉 [Azure Pipelines YAML schema - Extends](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops&pivots=templates-extends)

## © Copyright
Copyright (c) 2021-2026 Hector Frances

## 🤝 Contributing
Issues and pull requests are welcome! See the contribution guidelines (coming soon).

## 📜 License
This project is licensed under the terms of the [GNU General Public License v3.0](LICENSE).
