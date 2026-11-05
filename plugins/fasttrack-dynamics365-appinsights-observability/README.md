# FastTrack Dynamics 365 App Insights Observability

An instruction-based plugin for **Azure SRE Agent** named
`fasttrack-dynamics365-appinsights-observability`.

The plugin packages all 15 v2 skills from the
[Dynamics 365 observability recipe](../../recipes/dynamics-365-appinsights-observability/README.md).
Each skill retains its original name, `SKILL.md`, and all supporting files,
including query templates, interpretation guidance, and report contracts.

## Included skills

| Domain | Skill |
|---|---|
| Batch | `d365-batch-available-threads-analysis-v2` |
| Batch | `d365-batch-duration-execution-analysis-v2` |
| Batch | `d365-batch-task-heatmap-v2` |
| Batch | `d365-batch-task-priority-distribution-v2` |
| Batch | `d365-batch-throttling-analysis-v2` |
| Batch | `d365-batch-workload-overview-v2` |
| DMF | `d365-dmf-error-rca-v2` |
| DMF | `d365-dmf-execution-drilldown-v2` |
| DMF | `d365-dmf-execution-duration-analysis-v2` |
| DMF | `d365-dmf-monitoring-dashboard-v2` |
| DMF | `d365-dmf-throughput-volume-analysis-v2` |
| DMF | `d365-dmf-unfinished-runs-v2` |
| Forms | `d365-forms-form-drilldown-analysis-v2` |
| Forms | `d365-forms-regression-analysis-v2` |
| Forms | `d365-forms-usage-overview-v2` |

## Prerequisites

- An Azure SRE Agent in **Running** state and the **SRE Agent Author** or
  **Administrator** role.
- An Application Insights resource or Log Analytics workspace receiving
  Dynamics 365 Finance and Operations telemetry, with query access for the
  agent identity.
- The query and visualization tools declared in each skill's `SKILL.md`.
  Installing the plugin does not configure telemetry connections or permissions.
- The recipe's four required Agent Memory files, uploaded from their
  authoritative source and successfully indexed:
  - `d365-batch-heatmap-override-1h-buckets.md`
  - `knowledge_applicationinsightscustomevents-batch.json.json`
  - `knowledge_applicationinsightscustomevents-dmf.json.json`
  - `knowledge_applicationinsightsgeneralschema.json.json`

The Agent Memory contents are not present in the recipe export and are not
bundled in this plugin. Do not substitute placeholder knowledge files. Skills
also refer to logical schema names such as
`ApplicationInsightsGeneralSchema.json`,
`ApplicationInsightsCustomEvents-Batch.json`, and the DMF schema; ensure the
corresponding knowledge is retrievable before running an analysis.

## Install in Azure SRE Agent

The plugin must be committed and pushed to a GitHub repository accessible to
the agent before installation. It can be installed automatically by the
Dynamics 365 observability recipe, from this repository's marketplace, or
directly from its URL.

## Install with the recipe

The
[Dynamics 365 observability recipe](../../recipes/dynamics-365-appinsights-observability/README.md)
registers this repository's `fasttrack-dynamics365-observability` marketplace
and installs the plugin during deployment. This is the preferred option when
creating the complete agent because it also deploys the coordinator,
connectors, incident platform, and scheduled task.

If the repository is private, configure the agent's GitHub identity before
deployment. Do not put GitHub credentials in the recipe.

## Install directly

1. In the SRE Agent portal, open **Builder > Plugins > Install from URL**.
2. Set **GitHub repository URL** to
   `kenny-saelen/sre-d365-appinsights-observability-agent`.
3. Set **Path in repo** to
   `plugins/fasttrack-dynamics365-appinsights-observability`.
4. Use the portal's GitHub authentication options if the repository is private.
   Never store a personal access token in this repository.
5. Select **Install**, then verify that the success message reports **15 skills**.
6. Open **Builder > Skill builder** and confirm that all skills listed above
   were imported with their supporting files.

See Microsoft's
[Install a plugin from a URL](https://learn.microsoft.com/en-us/azure/sre-agent/install-plugin-from-url)
documentation. A marketplace registration is not required.

## Usage

Provide the target Application Insights resource or Log Analytics workspace,
the environment ID, and the UTC time range with the request. For example:

- "Analyze available batch threads for this environment over the last 24 hours."
- "Investigate DMF execution failures for this environment."
- "Compare form performance in this environment between these two time ranges."

The plugin imports skills only. It does not deploy the recipe's coordinator,
connectors, Azure resources, Agent Memory, incident platform, or scheduled task.
If you use an existing coordinator with an explicit skill allow-list, confirm
that it allows the imported skill names.

## Package maintenance

The plugin's `skills/*-v2/` directories are the source of truth. The recipe
references the plugin and must not contain copied skill directories or skill
manifests.

When skills change, update the matching plugin skill directories and increment
the plugin version. Include every supporting file and remove obsolete files.
Plugin discovery uses `skills/<skill-name>/SKILL.md`.
