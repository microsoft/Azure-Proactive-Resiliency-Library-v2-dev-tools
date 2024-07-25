---
title: Well-Architected Reliability Assessment (WARA) v2 collector script
weight: 10
geekdocCollapseSection: false
---

{{< toc >}}

# Well-Architected Reliability Assessment (WARA) v2 Collector Script

## Synopsis
Well-Architected Reliability Assessment (WARA) v2 collector script

## Description
This script is used to collect data from Azure subscriptions to be used in the Well-Architected Reliability Assessment (WARA) v2. The script collects data from the subscriptions, resource groups, and resources, and then runs resource graph queries (Kusto/KQL) to extract information about the resources. The script also collects information about outages, support tickets, advisor recommendations, service retirements, and service health alerts. The collected data is then used to generate a JSON file with recommendations for improving the reliability of the resources. Typically, this JSON file is used as an input for the WARA v2 data analyzer script (2_wara_data_analyzer.ps1).

By default, the script executes all relevant checks in the Azure Proactive Resiliency Library v2 but it can also be configured to run checks against specific groups of resources using a runbook (`-RunbookFile`).

## Requirements
- [PowerShell 7](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell?view=powershell-7.4)
- [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- [Azure PowerShell](https://learn.microsoft.com/en-us/powershell/azure/install-azps-windows?view=azps-12.1.0&tabs=powershell&pivots=windows-psgallery)
  - Az.ResourceGraph
  - Az.Accounts
- Reader access to resources to be evaluated

## Quick Start
- Download and run the collector script by copying and modifying this script block
- You must replace the TenantID and SubscriptionIds/ResourceGroups to match your tenant and scope resource ids.
- Once the script is downloaded you can execute it by running ./1_wara_collector.ps1
```powershell
#Download the latest version of the script
invoke-webrequest https://aka.ms/aprl/tools/1 -out 1_wara_collector.ps1
#Remove file blocking if active and running windows
$iswindows ? (unblock-file ./1_wara_collector.ps1) : (Write-host "Unblock not required - Not Windows OS")
#Modify these parameters and run the script
./1_wara_collector.ps1 -TenantID "00000000-0000-0000-0000-000000000000" -SubscriptionIds "/subscriptions/00000000-0000-0000-0000-000000000000"
```
### Local Machine
![Alt text](quickstartexample.gif)
### Cloud Shell
![Alt text](quickstartexample_cloudshell.gif)

## Filtering
The filtering capabilities are designed for targeting specific Azure resources, enabling precise and flexible reliability assessments. The scope of the feature includes functionalities that allow users to define the scope and tags and criteria of their reliability checks using parameters or text files.

### Order of operations

1. Subscriptions
   1. Subscription scopes like `-SubscriptionIds "/subscriptions/00000000-0000-0000-0000-000000000000"` or `[subscriptions]
/subscriptions/11111111-1111-1111-1111-111111111111` in a configuration file always take explicit precedence over any smaller, more specific scope.
2. Resource Groups
   1. These scopes can be used explicitly where you need to grab a resource group from a subscription but not evaluate the whole subscription.
3. Tags
   1. When your resources have been explicitly scoped as above - the script will then further refine your results based on the tags provided to the script via parameters or configuration file.

### Filtering Scenarios
- If you set a subscription filter for `subscription1` and you also set a resource group filter for `subscription1/resourcegroups/rg-demo1` your results will contain **all** of the resources in `subscription1`
  - This is because we specified `subscription1` and so all of `subscription1` will be evaluated. If we only wanted to evaluate `subscription1/resourcegroups/rg-demo1` then we would include that resource group as a filter and not the full subscription.
- If you set a subscription filter for `subscription2` and a resourcegroup filter for `subscription1/resourcegroups/rg-demo1` you will evaluate all of `subscription2` and only the resource group `rg-demo-1`.
- If you set a subscription filter for `subscription3`, a resourcegroup filter for `subscription1/resourcegroups/rg-demo1` and a tag filter for `env==prod` then you will **only return resources or resources that are in resource groups tagged with env==prod** within the scope of the evaluation which is `subscription3` and `subscription1/resourcegroups/rg-demo1`.

### Runbooks

Runbooks are JSON files that allow extensive customization of KQL queries executed by WARA v2 tooling and the resources these queries target. They also support the integration of custom KQL queries. Read on to learn more about using runbooks with WARA v2 tooling.

#### Selectors

Runbooks use selectors to identify groups of Azure resources for specific checks. [Selectors can be any valid KQL `where` condition.](https://learn.microsoft.com/azure/data-explorer/kusto/query/where-operator) Here are a few examples of valid runbook selectors:

| Pattern | Example | Notes |
| --- | --- | --- |
| By tag | `tags['app'] =~ 'my_app'` | Matches all resources tagged `app`: `my_app` |
| By regex pattern | `name matches regex '^my_app\d{2}$'` | Matches `my_app01`, `my_app02`, `my_app03` ... |
| By name | `name in~ ('my_app01', 'my_app02', 'my_app03')` | Matches only `my_app01`, `my_app02`, `my_app03` |
| By resource group | `resourceGroup =~ 'my_group'` | Matches all resources in the `my_group` resource group |
| By subscription | `subscriptionId =~ '1235ec12-...'` | Matches all resources in subscription `1235ec12-...` |

Each selector has a name (which can be referenced later in specific checks) and is defined in a runbook like this:

```json
{
  "selectors": {
    "my_app_resources": "tags['app'] =~ 'my_app'",
    "my_group_resources": "resourceGroup =~ 'my_group'",
    "my_app_and_my_group": "tags['app'] =~ 'my_app' and resourceGroup =~ 'my_group'"
  }
}
```

##### How selectors are applied to KQL queries

There are two different ways in which selectors can be applied to KQL queries:

* **Explicitly**: Including a `// selector` comment in your KQL query will automatically inject the appropriate selector condition at runtime.
  * For example, given the example selectors provided above, configuring a check to use the `my_app_resources` selector would automatically replace `// selector` with `| where tags['app'] =~ 'my_app'` at runtime. The best practice is to include `// selector` comments on their own line as, at runtime, they're automatically piped in as a `where` condition.
  * `// selector` is called the "default selector". You can also reference specific selectors in your KQL queries using this syntax: `// selector:name` where `name` is the name of the selector (e.g., `my_app_resource`). This is helpful when comparing different sets of resources.
* **Implicitly**: Most KQL queries including the default set included in APRL v2 don't include selector comments. Use the `-UseImplicitRunbookSelectors` script switch to automatically wrap every KQL query in an inner join that limits the scope of the query.

> __Important__: By default, implicit selectors will not be applied due to constraints on the total number of joins that can be included in a resource graph query. To use implicit selectors, you have to include the `-UseImplicitRunbookSelectors` when running the script.

Read on to learn how selectors and checks work together to run KQL queries against arbitrary groups of resources.

#### Checks

Checks combine selectors with specific KQL queries to run precise checks on arbitrary sets of resources. Here's an example using previously defined selectors:

```json
{
  "selectors": {
    "my_app_resources": "tags['app'] =~ 'my_app'",
    "my_group_resources": "resourceGroup =~ 'my_group'",
    "my_app_and_my_group_resources": "tags['app'] =~ 'my_app' and resourceGroup =~ 'my_group'"
  },
  "checks": {
    "122d11d7-b91f-8747-a562-f56b79bcfbdc": {
      "my_app_uses_managed_disks": "my_app_resources",
      "my_group_uses_managed_disks": "my_group_resources",
      "my_app_and_my_group_uses_managed_disks": "my_app_and_my_group_resources"
    }
  }
}
```

Let's break this down line by line:

* The `selectors` from the previous section are included to be used in the `checks` section of the runbook.
* A single check configuration is provided, using an existing APRL KQL query to verify that VM resources are using managed disks (`122d11d7-b91f-8747-a562-f56b79bcfbdc`).
* Three checks are defined, each applying to different selectors:
  * `my_app_uses_managed_disks`: Verifies that all resources with the tag `app` set to `my_app` use managed disks.
  * `my_group_uses_managed_disks`: Verifies that all resources in the `my_group` resource group use managed disks.
  * `my_app_and_my_group_uses_managed_disks`: Verifies that all resources in the `my_group` resource group with the tag `app` set to `my_app` use managed disks.

By combining checks and selectors, you can easily define complex WARA review configurations using a simple JSON-based syntax.

#### Parameters

Parameters offer a simple syntax for dynamically customizing selectors and KQL queries. Parameters are arbitrary key/value pairs that are included in the `parameters` section of a runbook like this:

```json
{
  "parameters": {
    "resource_group_name": "my_resource_group",
    "resource_tag_name": "my_tag",
    "resource_tag_value": "my_value"
  }
}
```

You can easily reference parameters using this syntax:

`{{parameter_name}}`

Parameters can easily be included in selectors like this:

```kusto
resourceGroup =~ '{{resource_group_name}}'
```

Parameters can also be included directly in queries like this:

```kusto
resources
| where tags[`{{resource_tag_name}}`] =~ '{{resource_tag_value}}`
```

# Examples for Well-Architected Reliability Assessment (WARA) v2 Collector Script

## Example 1
Run against one subscriptions in tenant `00000000-0000-0000-0000-000000000000`:
```powershell
.\1_wara_collector.ps1 -TenantID "00000000-0000-0000-0000-000000000000" -SubscriptionIds "/subscriptions/00000000-0000-0000-0000-000000000000"
```
## Example 2
Run against one subscription and one resource group in tenant `00000000-0000-0000-0000-000000000000`:
```powershell
.\1_wara_collector.ps1 -TenantID "00000000-0000-0000-0000-000000000000" -SubscriptionIds "/subscriptions/00000000-0000-0000-0000-000000000000" -ResourceGroups "/subscriptions/55555555-5555-5555-5555-555555555555/resourceGroups/Demo1-RG"
```
## Example 3
Run against two subscriptions and one resource group in tenant using tags to filter resources within the scope of the subscription and resource group in the tenant`00000000-0000-0000-0000-000000000000`:
```powershell
.\1_wara_collector.ps1 -TenantID "00000000-0000-0000-0000-000000000000" -SubscriptionIds "/subscriptions/00000000-0000-0000-0000-000000000000","/subscriptions/33333333-3333-3333-3333-333333333333" -ResourceGroups "/subscriptions/55555555-5555-5555-5555-555555555555/resourceGroups/Demo1-RG" -Tags 'Criticality==High','Env==Prod'
```
## Example 4
Run against one subscription and two resource groups in tenant `00000000-0000-0000-0000-000000000000` using tags to filter resources within the scope of the subscription and resource group in the tenant`00000000-0000-0000-0000-000000000000`:
```powershell
.\1_wara_collector.ps1 -TenantID "00000000-0000-0000-0000-000000000000" -SubscriptionIds "/subscriptions/00000000-0000-0000-0000-000000000000" -ResourceGroups "/subscriptions/55555555-5555-5555-5555-555555555555/resourceGroups/Demo1-RG","/subscriptions/44444444-4444-4444-4444-444444444444/resourceGroups/Demo2-RG" -Tags 'Criticality==High','Env==Prod'
```
**Note**: Multiple values do not have to be in the same subscription. You can specify multiple resource groups in unique subscriptions.
## Example 5
Run a configuration file:
```powershell
.\1_wara_collector.ps1 -ConfigFile ".\config.txt"
```
### ConfigFile Example
```text
[tenantid]
00000000-0000-0000-0000-000000000000

[subscriptions]
/subscriptions/11111111-1111-1111-1111-111111111111

[resourcegroups]
/subscriptions/55555555-5555-5555-5555-555555555555/resourceGroups/Demo1-RG
/subscriptions/22222222-2222-2222-2222-222222222222/resourceGroups/Demo1-RG

[tags]
env==prod
application==demoapp1
```
**Note**: In a configuration file we separate multiple entries for a filter by new lines. Where as, from the command line we would pass multiple subscriptions or resource groups using the "string1","string2" pattern. The configuration file is useful for repeated runs, or numerous filters where it may be difficult to troubleshoot syntax in the command line.

## Parameters

### Debugging
- **Type**: Switch
- **Description**: Enables debugging output.

### SAP
- **Type**: Switch
- **Description**: Enables recommendations and queries for the SAP specialized workload.

### AVD
- **Type**: Switch
- **Description**: Enables recommendations and queries for the AVD specialized workload.

### AVS
- **Type**: Switch
- **Description**: Enables recommendations and queries for the AVS specialized workload.

### HPC
- **Type**: Switch
- **Description**: Enables recommendations and queries for the HPC specialized workload.

### SubscriptionIds
- **Type**: String Array
- **Description**: Specifies the subscription IDs to be included in the review. Multiple subscription IDs should be separated by commas. Subscription IDs must be in either GUID form (e.g., `00000000-0000-0000-0000-000000000000`) or full subscription ID form (e.g., `/subscriptions/00000000-0000-0000-0000-000000000000`).
- **Note**: Can't be used in combination with `-ConfigFile` parameter.

### ResourceGroups
- **Type**: String Array
- **Description**: Specifies the resource groups to be included in the review. Multiple resource groups should be separated by commas. Resource groups must be in full resource group ID form (e.g., `/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg1`).
- **Note**: Can't be used in combination with `-ConfigFile` or `-RunbookFile` parameters.

### Tags
- **Type**: String
- **Description**: Specifies the tags to be used to filter resources.
- **Note**: Can't be used in combination with `-ConfigFile` or `-RunbookFile` parameters.

### TenantID
- **Type**: String
- **Description**: Specifies the Entra tenant ID to be used to authenticate to Azure.

### AzureEnvironment
- **Type**: String
- **Description**: Specifies the Azure environment to be used. Valid values are `AzureCloud` and `AzureUSGovernment`. Default value is `AzureCloud`.

### ConfigFile
- **Type**: String
- **Description**: Specifies the configuration file to be used.
- **Note**: Can't be used in combination with `-RunbookFile`, `-SubscriptionIds`, `-ResourceGroups`, or `-Tags` parameters.

### RunbookFile
- **Type**: String
- **Description**: Specifies the runbook file to be used. More information about runbooks:
  - The parameters section defines the parameters used by the runbook. These parameters will be automatically merged into selectors and queries at runtime.
  - The selectors section identifies groups of Azure resources that specific checks will be run against. Selectors can be any valid KQL predicate (e.g., `resourceGroup =~ 'rg1'`).
  - The checks section maps resource graph queries (identified by GUIDs) to specific selectors.
  - The query_overrides section enables catalogs of specialized resource graph queries to be included in the review.
- **Note**: Can't be used in combination with `-ConfigFile`, `-ResourceGroups`, or `-Tags` parameters. Specify subscriptions in scope using `-SubscriptionIds` parameter.

### UseImplicitRunbookSelectors
- **Type**: Switch
- **Description**: Enables the use of implicit runbook selectors. When this switch is enabled, each resource graph query will be wrapped in an inner join that filters the results to only include resources that match the selector. This is useful when queries do not include selectors.