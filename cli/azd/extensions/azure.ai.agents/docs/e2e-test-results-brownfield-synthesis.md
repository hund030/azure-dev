# Brownfield Synthesis E2E Test Results

Date: 2026-07-24

## Environment

- Platform: Windows
- Subscription: `Foundry Dev Tools - Dev Test with TTL = 3 Days`
- Local development `azd` and locally built `azure.ai.agents` and `azure.ai.projects` extensions
- Existing Foundry project: `pr9174-telemetry`
- Test environment: `bf-synth-e2e-0724`

All `azd` commands were run with `AZURE_DEV_USER_AGENT=microsoft_foundry_skill` set for that process.

## Build

From each extension directory:

```powershell
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd x build
```

Directories:

- `cli/azd/extensions/azure.ai.projects`
- `cli/azd/extensions/azure.ai.agents`

Result: both extensions built and installed successfully.

## Brownfield Eject Validation

The test `azure.yaml` set an existing project endpoint and included missing agent and connection `$ref` files.

```powershell
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd ai agent init --infra --no-prompt
```

Result: failed as expected with the stable brownfield-specific error:

```text
ERROR: `azd ai agent init --infra` is not supported for a project that reuses an existing Foundry resource (the azure.ai.project service sets endpoint:)
```

The missing sibling `$ref` files did not replace this error, confirming endpoint detection occurs before full sibling synthesis on the eject path.

## Environment Setup

```powershell
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd env new bf-synth-e2e-0724 `
  --subscription 1756abc0-3554-4341-8d6a-46674962ea19 `
  --location northcentralus `
  --no-prompt

$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd env set AZURE_AI_PROJECT_ID `
  '/subscriptions/1756abc0-3554-4341-8d6a-46674962ea19/resourceGroups/rg-pr9174-telemetry/providers/Microsoft.CognitiveServices/accounts/cog-qzjgxfl6754u6/projects/pr9174-telemetry'

$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd env set AI_AGENT_PENDING_PROVISION ''
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd env set AZURE_CONTAINER_REGISTRY_ENDPOINT ''
```

Result: the disposable local environment was created successfully with no pending ACR work.

## Provision Preview

```powershell
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd provision --preview --no-prompt
```

Result:

```text
Using existing Foundry project (endpoint set); nothing to provision
SUCCESS: Generated provisioning preview in 3 seconds.
```

This confirms the provider selected the brownfield synthesis path and produced an empty plan when no managed deployments, connections, or ACR were requested.

## Provision

```powershell
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd provision --no-prompt --output json
```

Result: provision completed successfully in approximately two seconds and returned:

```json
{
  "outputs": {
    "AZURE_AI_PROJECT_NAME": {
      "type": "string",
      "value": "pr9174-telemetry"
    },
    "FOUNDRY_PROJECT_ENDPOINT": {
      "type": "string",
      "value": "https://cog-qzjgxfl6754u6.services.ai.azure.com/api/projects/pr9174-telemetry"
    }
  },
  "resources": []
}
```

The environment values were also verified with:

```powershell
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd env get-values
```

No matching resource-group ARM deployment was created:

```powershell
az deployment group list `
  --resource-group rg-pr9174-telemetry `
  --query "[?contains(name, 'bf-synth-e2e-0724')].{name:name,state:properties.provisioningState}" `
  -o json
```

Result:

```json
[]
```

## Down Safety

```powershell
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd down --force --no-prompt
```

Result: completed successfully in less than one second.

The existing Foundry project was independently verified afterward:

```powershell
az resource show `
  --ids '/subscriptions/1756abc0-3554-4341-8d6a-46674962ea19/resourceGroups/rg-pr9174-telemetry/providers/Microsoft.CognitiveServices/accounts/cog-qzjgxfl6754u6/projects/pr9174-telemetry' `
  --api-version 2025-06-01 `
  --query id `
  -o tsv
```

Result: the existing project resource ID was returned, confirming `azd down --force` did not delete it.

## Cleanup

```powershell
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd env remove bf-synth-e2e-0724 --force --no-prompt
```

Result: the local azd environment and all scratch project files were removed. No Azure resources were created by these tests.

## Not Covered

The initial run above deliberately avoided modifying a shared project. A second
run created a fully disposable Foundry project and exercised resource-creating
brownfield behavior, as recorded below.

## Disposable Resource-Creating Run

### Greenfield Project Creation

Disposable resources:

- Environment: `bf-synth-7peig9zu`
- Resource group: `rg-bf-synth-7peig9zu`
- Foundry account: `cog-euvcs2fs6nhtm`
- Foundry project: `bf-synth-7peig9zu`
- Region: West US 2

The greenfield project was created with:

```powershell
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd env new bf-synth-7peig9zu `
  --subscription 1756abc0-3554-4341-8d6a-46674962ea19 `
  --location westus2 `
  --no-prompt

$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd env set AZURE_RESOURCE_GROUP rg-bf-synth-7peig9zu
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd env set AZURE_AI_PROJECT_NAME bf-synth-7peig9zu
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd provision --no-prompt --output json
```

The first attempt in North Central US failed at Azure preflight with
`InsufficientQuota`. It created only an empty resource group, which was deleted
before retrying in West US 2.

The West US 2 Azure deployment succeeded and created the account and project.
The initially installed development `azd` then panicked while refreshing
extension state with `unknown provisioning.ParameterType value: String`. This
was outside the extensions under test. A fresh core `azd` was built from the
workspace and used for the remaining commands.

### Brownfield Configuration

The disposable project was then referenced through `endpoint:` from a separate
azd environment. Its `azure.yaml` declared:

- Model deployment `gpt-4o-mini-e2e`
- `gpt-4o-mini`, version `2024-07-18`
- `GlobalStandard`, capacity `1`
- Project connection `test-connection`
- Pending ACR reason through `AI_AGENT_PENDING_PROVISION=acr`

### Brownfield Preview With ACR

```powershell
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd provision --preview --no-prompt
```

Result:

```text
Skip   : Azure AI Services                  : cog-euvcs2fs6nhtm
Create : Azure AI Services Model Deployment : gpt-4o-mini-e2e
Skip   : Foundry project                    : cog-euvcs2fs6nhtm/bf-synth-7peig9zu
Create : Foundry project connection         : acr7ddcab7a-conn
Create : Foundry project connection         : test-connection
Create : Container Registry                 : acr7ddcab7a
```

This confirms synthesized brownfield resources were passed to the embedded
template while the existing account and project remained references.

### ACR Permission Limitation

The first apply created the model deployment and ACR but failed while creating
the `AcrPull` role assignment because the test identity lacked
`Microsoft.Authorization/roleAssignments/write`.

The ACR itself was independently verified as `Succeeded`:

```text
acr7ddcab7a.azurecr.io
```

This validates ACR synthesis and creation, but not successful ACR RBAC or the
ACR project connection in this subscription.

### Model And Connection Provision

After clearing only the pending ACR signal, provisioning succeeded:

```powershell
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd env set AI_AGENT_PENDING_PROVISION ''
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd provision --no-prompt --output json
```

The following resources were independently verified:

```text
Model deployment: gpt-4o-mini-e2e
State: Succeeded
Model: gpt-4o-mini
Version: 2024-07-18
Capacity: 1

Project connection: test-connection
Container registry: acr7ddcab7a
ACR state: Succeeded
```

A second `azd provision --no-prompt` also succeeded, validating safe repeated
upsert behavior. ARM what-if reported `Modify` for the model deployment and
connection rather than `NoChange`, so idempotency was established through the
successful repeat apply.

### Brownfield Down Safety

```powershell
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd down --force --no-prompt
```

Result:

```text
Foundry project is bring-your-own (endpoint set); azd did not create it, so azd down leaves it in place
```

The project, model deployment, and `test-connection` were all independently
verified to still exist after this command.

### Final Cleanup

The original greenfield environment deleted the complete disposable resource
group and purged the Cognitive Services account:

```powershell
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'; azd down --force --purge --no-prompt
```

Result: completed successfully in approximately four minutes.

Independent cleanup checks:

```powershell
az group exists --name rg-bf-synth-7peig9zu
az cognitiveservices account list-deleted `
  --query "[?name=='cog-euvcs2fs6nhtm'].{name:name,location:location}" `
  -o json
```

Results:

```text
false
[]
```

Both local azd environments and all scratch directories were also removed.
