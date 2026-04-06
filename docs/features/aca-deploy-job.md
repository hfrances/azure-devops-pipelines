# ACA Deploy Job

Template: `jobs-templates/aca-deploy-job.yml`

## What It Does

It deploys an Azure Container App from an existing ACR image.

It can also manage the target Container Apps Environment when the caller enables that mode.

## Inputs

| Param | Default | Meaning |
| --- | --- | --- |
| `jobName` | `DeployToACA` | generated job name |
| `displayName` | `Deploy to ACA` | generated job display name |
| `dependsOn` | `[]` | optional job dependencies |
| `condition` | `''` | optional job condition |
| `variables` | `{}` | optional job-level variables |
| `azureServiceConnection` |  | Azure service connection |
| `resourceGroup` |  | resource group for app, identity, and CAE |
| `identityName` |  | user-assigned identity used for ACR pull |
| `acrName` |  | ACR name without `.azurecr.io` |
| `imageRepository` |  | repository inside ACR |
| `imageTag` |  | image tag to deploy |
| `caeManaged` | `false` | whether the template manages the CAE |
| `caeName` |  | target CAE name |
| `caeLocation` | `''` | location used only when `caeManaged=true` |
| `acaName` |  | target Container App name |
| `acaIngressType` | `external` | `internal` or `external` |
| `acaTargetPort` | `80` | target port exposed by the app |
| `acaCpu` | `0.5` | app CPU |
| `acaMemory` | `1.0Gi` | app memory |
| `acaMinReplicas` | `0` | minimum replicas |
| `acaMaxReplicas` | `1` | maximum replicas |
| `envVars` | `[]` | env vars in `NAME=value` format |
| `useSubnet` | `false` | private CAE mode when `caeManaged=true` |
| `resourceGroupVNet` | `''` | VNet resource group for private CAE mode |
| `vnetName` | `''` | VNet name for private CAE mode |
| `subnetName` | `''` | subnet name for private CAE mode |

## Outputs

The deploy step exports:

- `deploy.acaFqdn`
- `deploy.acaHost`

Use them from downstream jobs with:

```yml
$[ dependencies.<job>.outputs['deploy.acaFqdn'] ]
$[ dependencies.<job>.outputs['deploy.acaHost'] ]
```

## CAE Management Modes

- `caeManaged=false`
  - the CAE must already exist
  - the template does not validate or manage CAE network settings
- `caeManaged=true` and `useSubnet=false`
  - the template creates or validates a public CAE
- `caeManaged=true` and `useSubnet=true`
  - the template validates the subnet
  - if the subnet is empty, it applies `Microsoft.App/environments` delegation
  - the template creates or validates a private CAE bound to that subnet

## App Deployment Behavior

The app step:

- resolves the managed identity
- assigns the identity to the app
- configures ACR access through that identity
- creates the app if it does not exist
- updates the app if it already exists
- re-applies ingress and validates the final state

## Warnings and Failure Mode

The job fails when:

- ingress type is invalid
- required CAE inputs are missing in managed mode
- subnet validation fails in private managed mode
- CAE creation or validation fails in managed mode
- deployed ingress does not match the requested ingress mode

The job completes as `SucceededWithIssues` when:

- `acaIngressType=external`
- Azure reports external ingress
- but the app FQDN is still empty
The job also prints a hosts-file warning only when:

- `useSubnet=true`
- `acaIngressType=external`
- both CAE static IP and app FQDN are available

