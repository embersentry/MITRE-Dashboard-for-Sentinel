# MITRE-Dashboard-for-Sentinel

This dashboard is to be used as a way to automatically track the analytics that have been deployed on a workspace.

## What is here

| File | Purpose |
|---|---|
| `ATTCK_Coverage_Board (2).html` | The dashboard. Open it in Chrome or Edge, connect it to `board.json` and it renders the ATT&CK matrix from that file. |
| `board (4).json` | The shared board data: customer views, and per view a map of technique ID to the analytics mapped onto it. |
| `azuredeploy.json` | Logic App that reads one Sentinel workspace's analytics rules nightly and writes them into `board.json`. |
| `azuredeploy.parameters.json` | The values to change per customer. |

## What the Logic App does

One deployment per customer. Each night it:

1. Lists every analytics rule in the workspace (`workspaceResourceId`) over the management API, paging through `nextLink`, authenticating with the Logic App's own managed identity.
2. Shapes each rule into a board entry (`Shape_entries`) with the same fields the dashboard writes itself, plus `source: "sentinel"` so the sync can recognise its own work later.
3. Groups those entries by the techniques each rule is tagged with, pushing revoked ATT&CK IDs through `techniqueRemap` so they land on tiles the dashboard's v19.2 matrix still draws (`Merge_into_board`).
4. Replaces the previous sync's entries in the customer's view with that set, leaving anything added by hand in the dashboard untouched.
5. Writes the file back to SharePoint — unless `dryRun` is `true`, in which case it stops and you read the `Summary` step in run history.

Repeat runs are idempotent: entry counts stay flat, and a rule deleted in Sentinel loses its entries on the next run.

## Deploying

1. Fill in `azuredeploy.parameters.json`. Every parameter marked **CHANGE** in the template's descriptions needs a real value.
2. Deploy to the resource group that should own the Logic App. `assignSentinelReader` grants the managed identity Microsoft Sentinel Reader on the workspace, which needs Owner or User Access Administrator there.
3. **Authorise the SharePoint connection.** The deployment creates it but cannot sign it in. Open the connection named in the `sharePointConnectionToAuthorise` output and authenticate, or the first run fails on `Get_board_metadata`.
4. Leave `dryRun` at `true` for the first run, check the `Summary` step, then set it to `false`.

### Integration account

The merge step is an Execute JavaScript Code action, which a Consumption Logic App only runs when the workflow is linked to an integration account — hence `createIntegrationAccount`. The Free tier allows **one per region per subscription**, so for the second and later customers in a region set `createIntegrationAccount` to `false` and point `existingIntegrationAccountId` at the first one. That parameter is required in that case; left empty the deployment fails.

That same action caps the code at **1,024 characters** on Consumption, which is why the entry shaping lives in the `Shape_entries` action rather than in the script. Keep the script under that limit or move the workflow to a Standard logic app, where the cap is 100,000.

## Known limits

- **Sub-techniques depend on the API version.** Stable `Microsoft.SecurityInsights` versions, including the default `2024-09-01`, do not return `subTechniques`, so rules map onto parent technique tiles only. A preview version that returns it, such as `2024-10-01-preview`, maps them onto sub-technique tiles too.
- **Techniques outside the dashboard's matrix are stored but not drawn.** `techniqueRemap` covers every revoked ID, but a rule tagged with an ID MITRE retired without a replacement — `T1026`, `T1034`, `T1043`, `T1051`, `T1053.004`, `T1061`, `T1062`, `T1064`, `T1108`, `T1149`, `T1153`, `T1175` — or with a technique newer than the matrix, lands under a key the grid never renders. Re-tag those rules in Sentinel.
- **Synced rules show as mapped but unlinked.** The sync cannot know a OneNote page, so `url` is empty and those tiles count as partial coverage, not full.
- **Editing a synced entry in the dashboard makes it permanent.** The edit form rewrites the entry from its own fields, dropping `source`, so later runs treat it as hand-added: its status is no longer refreshed from Sentinel and it is not removed when the rule is deleted there.
- **The dashboard can resurrect deleted entries.** When it finds a newer `board.json` it offers Merge / Use the file / Keep mine, and Merge is additive — it will bring back entries the sync removed. After a sync, prefer **Use the file**, or **Pull** before editing.
