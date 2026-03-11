# 📦 Azure-Devops-Pipelines (Reusable Pipeline Templates)

This repository contains a collection of reusable **Azure Pipelines YAML templates** designed to standardize and simplify CI/CD workflows across multiple projects.

## 🎯 Purpose

These templates are intended to be imported into project-specific pipelines using the `resources.repositories` and `extends` mechanisms provided by Azure DevOps. This allows teams to centralize common logic, enforce best practices, and reduce duplication.

## 🚀 How to Use

In your project’s pipeline (`azure-pipelines.yml`), reference this repository like so:

```yaml
resources:
  repositories:
    - repository: azure-devops-pipelines
      type: git
      name: https://github.com/hfrances/azure-devops-pipelines
      ref: refs/heads/master  # or a specific tag/branch

extends:
  template: standard/azure-pipelines-template-aspnet.yml@azure-devops-pipelines
  parameters:
    param1: value1
    param2: value2
```

## 📚 Template Parameters

Each template contains a `parameters:` block at the top that defines the inputs it expects. Check the header comments or accompanying `README.md` inside each template folder for usage examples.

## 📖 Official Documentation

For more details on how to structure and use templates in Azure Pipelines, refer to the official Microsoft documentation:  
👉 [Azure Pipelines YAML schema - Extends](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops&pivots=templates-extends)

## © Copyright
Copyright (c) 2021-2026 Hector Frances

## 🤝 Contributing
Issues and pull requests are welcome! See the contribution guidelines (coming soon).

## 📜 License
This project is licensed under the terms of the [GNU General Public License v3.0](LICENSE).
