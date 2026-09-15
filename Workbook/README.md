# Azure Workbook templates

This folder contains two versions of the Monthly SLA Reporting Azure Workbook:

| Template | Use case |
| --- | --- |
| `workbook.json` | Standard template for reporting across any subscriptions selected manually by the user. |
| `workbook - with Environment tag filter.json` | Newer template for tenants that classify subscriptions with an `Environment` tag. |

The newer template adds a required global **Environment** filter with the values `Production`, `Test`, `Development`, and `Hybrid`. The selected value is used to find subscriptions whose `Environment` tag matches it; those subscriptions then scope the available Log Analytics workspaces and the resulting SLA reporting data.

Use the Environment-filtered template when a tenant has multiple environments and needs to report on one environment at a time, such as production-only SLA reporting. Use the standard template when subscriptions are not consistently tagged, or when manually selecting an arbitrary cross-environment subscription set is required.