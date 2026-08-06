# Azure Workbook for SLA Reporting

This repository contains an Azure Monitor Workbook that calculates an Azure Resource Health availability baseline across selected subscriptions, Log Analytics workspaces, and reporting periods.

The workbook presents:

- A summary of availability percentages by workload category.
- Detailed tables showing resource count, eligible minutes, unavailable minutes, and availability.
- Global filters for subscriptions, Log Analytics workspaces, and 30-, 60-, or 90-day reporting periods.
- Export-to-Excel support for the detailed tables.

> [!IMPORTANT]
> The workbook reports availability derived from Azure Resource Health state transitions in the Azure Activity Log. 
> It is an operational availability baseline, not a contractual Azure SLA calculation, service-credit determination, or end-to-end application uptime measurement.


Screenshot 01:
![Screenshot 01](Images/Screenshot--01.jpeg)


Screenshot 02:
![Screenshot 02](Images/Screenshot--02.jpeg)


## Repository structure

```text
.
|-- README.md
`-- Workbook
    `-- workbook.json
`-- Images
    `-- Screenshot--01.jpeg
    `-- Screenshot--02.jpeg
```

`Workbook/workbook.json` contains the Azure Workbook definition and embedded KQL queries.

## Workload coverage

| Workbook category | Azure resource types | Measurement |
| --- | --- | --- |
| Compute | `Microsoft.Compute/virtualMachines`, `Microsoft.Compute/virtualMachineScaleSets` | Direct Resource Health state |
| Storage | `Microsoft.Storage/storageAccounts` | Direct Resource Health state |
| Disks | `Microsoft.Compute/disks` | Attached VM Resource Health used as a proxy |
| SQL databases | `Microsoft.Sql/servers/databases`, `Microsoft.Sql/managedInstances` | Direct Resource Health state |
| Apps/Web | `Microsoft.Web/sites` | Direct Resource Health state |

The Apps/Web category includes App Service, Azure Functions, and Logic Apps Standard. Consumption Logic Apps use `Microsoft.Logic/workflows` and are not included because they do not provide the same Resource Health signal.

App Service Environments are not counted as separate resources. Applications hosted in an App Service Environment are included through their individual `Microsoft.Web/sites` resources.

## Workbook output

Each detailed workload query returns one aggregate row:

| Column | Description |
| --- | --- |
| `Category` | Workload category |
| `Resources` | Number of distinct resources included |
| `EligibleMinutes` | Total resource-minutes in the reporting period |
| `UnavailableMinutes` | Resource-minutes where Resource Health reported `Unavailable` |
| `Availability` | Aggregate availability percentage, rounded to four decimal places |

For example, three resources over a 60-day period have:

```text
3 resources x 60 days x 24 hours x 60 minutes = 259,200 eligible minutes
```

Availability is calculated as:

```text
Availability % =
100 x (Eligible minutes - Unavailable minutes) / Eligible minutes
```

Because all currently discovered resources are assigned the same reporting period, this pooled resource-minute calculation is also the average availability across the resources.

Only the `Unavailable` state is subtracted. `Degraded` and `Unknown` transitions are retained in the reconstructed timeline but are not currently counted as unavailable minutes.

## Prerequisites

### Azure Activity Log export

The selected subscriptions must export their Azure Activity Log to at least one selected Log Analytics workspace. The export must include the `ResourceHealth` category.

Resource Health records appear in the `AzureActivity` table and describe changes in Azure's health assessment for individual resources.

Common states are:

| State | Meaning |
| --- | --- |
| `Available` | Azure detects no issue affecting the resource |
| `Unavailable` | Azure detects that the resource cannot provide its expected service |
| `Degraded` | The resource remains available but has reduced performance or functionality |
| `Unknown` | Azure does not currently have enough health information |

### Permissions

Workbook users require:

- Read access to the selected subscriptions and resources so Azure Resource Graph can build the inventory.
- Query access to the selected Log Analytics workspaces, such as the **Log Analytics Reader** role.
- Access to the workbook resource itself.

### Log Analytics workspace scope

The `LogAnalyticsWorkspace` parameter supports multiple selections. Each Logs query is scoped through the workbook's `crossComponentResources` setting:

```json
"crossComponentResources": [
  "{LogAnalyticsWorkspace}"
]
```

The KQL therefore references:

```kusto
AzureActivity
```

rather than:

```kusto
workspace({LogAnalyticsWorkspace}).AzureActivity
```

`workspace()` accepts one workspace expression and cannot consume the comma-separated values produced by a multi-select parameter. Workbook resource scoping allows Azure Monitor to query all selected workspaces collectively.

## Global parameters

The workbook defines three global filters.

### `subscriptionID`

A required, multi-select subscription parameter. Queries compare the selected ARM subscription paths to the subscription GUIDs in Azure Resource Graph and `AzureActivity` by normalizing them:

```kusto
where strcat("/subscriptions/", subscriptionId) in~ ({subscriptionID})
```

The `AzureActivity` schema uses an uppercase column name:

```kusto
where strcat("/subscriptions/", SubscriptionId) in~ ({subscriptionID})
```

This case difference is intentional.

### `LogAnalyticsWorkspace`

A required, multi-select resource parameter populated from Azure Resource Graph:

```kusto
resources
| where type == "microsoft.operationalinsights/workspaces"
| project id
```

It controls the workspace scope for all downstream Logs queries.

### `TimeRange`

A required time-range parameter with:

- Last 30 days
- Last 60 days
- Last 90 days

Queries use the parameter boundaries directly:

```kusto
let StartTime = {TimeRange:start};
let EndTime = {TimeRange:end};
```

## How the KQL works

All workload queries follow the same general pipeline.

### 1. Define the reporting interval

```kusto
let StartTime = {TimeRange:start};
let EndTime = {TimeRange:end};
```

The calculation uses exact parameter timestamps rather than assuming calendar-month boundaries.

### 2. Read Activity Log data

```kusto
let SourceActivity =
    AzureActivity
    | where strcat("/subscriptions/", SubscriptionId) in~ ({subscriptionID})
    | where TimeGenerated between ((StartTime - 90d) .. EndTime);
```

The extra 90-day lookback finds the last known state before `StartTime`. This matters when a resource became unavailable before the selected reporting period and recovered during it.

### 3. Validate that the selected workspace scope has data

```kusto
let ActivityRows = toscalar(
    SourceActivity
    | summarize count());
```

The final pipeline includes:

```kusto
| where ActivityRows > 0
```

Without this guard, every inventory resource would retain its seeded `Available` state and an empty workspace could incorrectly produce 100% availability. With the guard, no result row is returned when the selected workspace scope contains no matching `AzureActivity` records for the selected subscriptions and lookback period.

This check proves that Activity Log data exists in the selected scope. It does not prove that every resource emitted a Resource Health event.

### 4. Discover resources with Azure Resource Graph

Each query uses:

```kusto
arg("").Resources
```

to build a current resource inventory. The resource-type filter changes by workload category.

The inventory then uses a small cross-service bridge:

```kusto
| extend JoinKey = 1
| join hint.remote=right kind=inner (
    print JoinKey = 1
) on JoinKey
```

This forces subsequent processing onto the Log Analytics side. It is required because operators later in the query, including `serialize`, are not supported by Azure Resource Graph. Without the bridge, Azure Monitor can push too much of the query to Resource Graph and return a parser failure.

### 5. Normalize Resource Health events

Queries filter:

```kusto
CategoryValue in~ ("ResourceHealth", "Resource Health")
```

They support both the newer dynamic `Properties_d` field and the older string `Properties` field:

```kusto
extend P = iif(
    isnotempty(tostring(Properties_d)),
    Properties_d,
    parse_json(Properties))
```

The current state is normalized across schema variants:

```kusto
extend State = tostring(
    coalesce(
        P.currentHealthStatus,
        P.healthStatus))
```

Resource IDs are converted to lowercase so Resource Graph inventory IDs and Activity Log IDs join consistently.

### 6. Seed and reconstruct each timeline

Each inventory resource receives a baseline row:

```kusto
EventTime = StartTime,
State = "Available",
Priority = 0
```

If a Resource Health event exists before the reporting period, its latest state overrides the baseline with priority `1`. Events inside the reporting period have priority `2`.

```kusto
| summarize arg_max(Priority, State) by ResourceId, EventTime
```

After sorting each resource's transitions, `next()` determines when the current state ends:

```kusto
| sort by ResourceId asc, EventTime asc
| serialize
| extend
    NextResourceId = next(ResourceId),
    NextEventTime = next(EventTime)
```

The interval ends at the next event for the same resource or at `EndTime`.

### 7. Aggregate resource-minutes

```kusto
| summarize
    Resources = dcount(ResourceId),
    EligibleMinutes = sum(IntervalMinutes),
    UnavailableMinutes = sumif(
        IntervalMinutes,
        State =~ "Unavailable")
```

The summary section uses global parameters such as `Availability_VM` and `Availability_SQL`, which run equivalent queries but project only `Availability`. The detailed section projects all five output columns.

The summary table displays **Above 99.9%** as a static reference target for every category. The workbook does not derive that target from resource SKU, topology, redundancy, availability-zone configuration, or the applicable Azure service terms. Replace the displayed target where a workload has a different contractual SLA.

## Differences between workload queries

The timeline and aggregation logic is shared. The important differences are inventory selection, event filtering, and resource normalization.

### Virtual machines and VM Scale Sets

Inventory types:

```kusto
where type in~ (
    "microsoft.compute/virtualmachines",
    "microsoft.compute/virtualmachinescalesets")
```

The event filter distinguishes standalone VMs from VM Scale Sets and prevents child instance paths from being counted accidentally.

Each VM Scale Set is counted as one resource. Individual VMSS instances are not counted separately.

### Storage accounts

Inventory type:

```kusto
where type =~ "microsoft.storage/storageaccounts"
```

The Resource Health filter targets the storage-account resource path. Blob, File, Queue, and Table services are represented through their parent storage account rather than counted as separate resources.

Managed disks are not included in this category because they are `Microsoft.Compute/disks` resources and do not expose an equivalent independent Resource Health availability signal.

### Managed disks

Inventory type:

```kusto
where type =~ "microsoft.compute/disks"
| where isnotempty(managedBy)
```

Only attached disks are included. `managedBy` identifies the associated VM:

```kusto
VmId = tolower(tostring(managedBy))
```

Managed disks do not provide independent Resource Health availability events. The disk query therefore:

1. Reads Resource Health transitions for standalone VMs.
2. Joins each attached disk to its managing VM.
3. Applies the VM state to the disk timeline.

The result is labelled `Managed disks (VM proxy)` because it represents host-VM availability, not a direct disk SLA or independent disk-health measurement.

### Azure SQL Database and SQL Managed Instance

Inventory types:

```kusto
where type in~ (
    "microsoft.sql/servers/databases",
    "microsoft.sql/managedinstances")
```

The query excludes Azure SQL Database `master` databases:

```kusto
where not(
    type =~ "microsoft.sql/servers/databases"
    and name =~ "master")
```

For SQL Managed Instance, the managed instance is counted at the parent resource level. Child MI databases are excluded to prevent one managed instance from being counted once for the instance and again for every database.

Azure SQL Database is counted at the database resource level because Resource Health supports `Microsoft.Sql/servers/databases`.

### App Service, Functions, and Logic Apps Standard

Inventory type:

```kusto
where type =~ "microsoft.web/sites"
```

All three workloads share `Microsoft.Web/sites`. The `kind` property classifies them:

```kusto
WorkloadType = case(
    tolower(tostring(kind)) contains "functionapp",
        "Azure Functions",
    tolower(tostring(kind)) contains "workflowapp",
        "Logic Apps Standard",
    "App Service")
```

Deployment slots are excluded:

```kusto
where ResourceIdLower !contains "/slots/"
```

This avoids counting both a production app and each of its slots as separate top-level applications.

Consumption Logic Apps are not included because they use `Microsoft.Logic/workflows`. App Service Environment resources are also not counted separately, although apps hosted in an environment remain included as `Microsoft.Web/sites`.

## Assumptions and limitations

### Resource Health is not end-to-end uptime

Resource Health represents Azure's assessment of a resource. It does not directly measure whether users could complete a business transaction or reach an application endpoint. Use Application Insights availability tests or another synthetic-monitoring solution for customer-observed uptime.

### The initial state defaults to available

If no earlier transition is found in the 90-day lookback, the resource is assumed to be available at `StartTime`. This is necessary because Resource Health produces transitions rather than periodic heartbeat rows, but it can overstate availability when history is incomplete.

### Current inventory is used

Azure Resource Graph supplies the inventory at query time. As a result:

- A resource deleted before the workbook runs is absent even if it existed during the reporting period.
- A newly created resource can be treated as eligible for the entire selected period.

For strict historical asset eligibility, creation and deletion timestamps must be stored and incorporated separately.

### Empty data and missing tables are different

If the selected workspace scope contains `AzureActivity` but no matching health transition for a resource, that resource retains the seeded available state.

If no relevant `AzureActivity` rows exist, `ActivityRows > 0` suppresses the output. If the `AzureActivity` table does not exist at all in a selected query scope, Azure Monitor can return a table-resolution error before the guard executes.

### Activity Log retention and collection start

The workbook cannot report events that were never exported or that are outside available retention. Configure Activity Log export before relying on the report, and retain the data for at least the maximum reporting period plus the 90-day state lookback.

### Short and unknown transitions

Not every Resource Health transition is recorded in the Activity Log. For example, some short VM unhealthy periods and certain transitions to or from `Unknown` can be omitted.

### Only `Unavailable` reduces availability

The current queries do not subtract `Degraded` or `Unknown` time. Change the `sumif` predicate if a different organizational policy is required.

### Managed disks are a proxy

The managed-disk result inherits the state of the attached standalone VM. It does not independently prove disk availability, and disks whose `managedBy` resource does not match the standalone VM event model do not receive correlated health transitions.

### Azure Resource Graph cross-service limit

The `arg()` cross-service query returns at most the first 1,000 Resource Graph records. The workbook is intended for scopes below that limit.

## Import the workbook

1. Open **Azure Monitor** in the Azure portal.
2. Select **Workbooks** and create a new workbook.
3. Open the workbook's advanced editor.
4. Replace the template JSON with the contents of `Workbook/workbook.json`.
5. Apply the changes and save the workbook.
6. Select the subscriptions, Log Analytics workspaces, and reporting period.

If a category returns no row, confirm that:

1. At least one selected workspace contains `AzureActivity`.
2. The selected subscriptions export Activity Log data to the selected workspace scope.
3. The user has query permission on every selected workspace.
4. The selected subscription and workspace parameters are populated before downstream queries run.

## Interpretation guidance

Use this workbook to:

- Establish a consistent Resource Health availability baseline.
- Compare broad workload categories over the same reporting period.
- Identify whether Azure reported unavailable time for selected resources.
- Export aggregate results for operational reporting.

Do not use the workbook alone to:

- Determine Azure service-credit eligibility.
- Replace service-specific SLA terms.
- Prove end-user or application availability.
- Measure independent managed-disk health.
- Reconstruct an authoritative historical inventory.

For formal reporting, label the result **Azure Resource Health availability** and retain the selected subscriptions, workspaces, reporting period, query version, and workbook version with the exported result.
