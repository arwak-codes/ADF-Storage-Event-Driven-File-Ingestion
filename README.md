# Azure Data Factory: Event-Driven File Ingestion

An Azure Data Factory (ADF) pipeline that automatically moves files from one Blob Storage container to another the moment they are uploaded. No manual runs, no schedules. A **Storage Event Trigger** starts the pipeline whenever a new blob lands in the source container.

![Pipeline design](Pipeline%20Design.png)

## What it does

Upload a file to `triggersource` and, within a minute or so:

1. The storage event trigger fires and starts the pipeline.
2. **Get Metadata** lists the files in the source container.
3. **ForEach** loops over each file.
4. **Copy Data** copies the file to `triggerdestination`.
5. **Delete** removes the file from `triggersource` once the copy succeeds.

The result is a simple "move file" pattern: files end up in the destination container and the source container stays clean.

## Architecture

```
Blob uploaded to triggersource
            │
            ▼
   Storage Event Trigger
            │
            ▼
      Get Metadata  (Child Items)
            │
            ▼
         ForEach  (@activity('Get Metadata').output.childItems)
            │
            ├──► Copy Data       (triggersource ──► triggerdestination)
            │         │ on success
            └──► DeleteCopiedFile (removes the file from triggersource)
```

## Azure services used

| Service | Purpose |
|---|---|
| Azure Data Factory | Orchestration, trigger, and pipeline |
| Azure Blob Storage | Source and destination containers |

## Pipeline details

| Item | Value |
|---|---|
| Pipeline name | `Practice_Foreach_Copy_Delete` |
| Data factory | `ADFarwa` |
| Trigger type | Storage event trigger (blob created) |
| Source container | `triggersource` |
| Destination container | `triggerdestination` |
| Integration runtime | AutoResolveIntegrationRuntime (Central India) |

### Activities

| Activity | Type | Role |
|---|---|---|
| `Get Metadata` | Get Metadata | Reads the **Child Items** of the source container to get the list of files |
| `ForEach` | ForEach | Iterates over the file list returned by Get Metadata |
| `Copy Data` | Copy data | Copies the current file (`@item().name`) to the destination |
| `DeleteCopiedFile` | Delete | Deletes the current file from the source after a successful copy |

## Key concepts practiced

- Storage event triggers (event-driven vs. scheduled pipelines)
- Dynamic content and expressions
- Dataset parameters (passing the file name into a reusable dataset)
- `Get Metadata` with the **Child Items** field
- `ForEach` loops and the `@item().name` expression
- Chaining activities on success dependencies

## How to recreate it

1. **Create two containers** in a storage account: `triggersource` and `triggerdestination`.
2. **Create linked services** in ADF for the storage account.
3. **Create datasets** for the source and destination. Add a `fileName` parameter to the dataset used in Copy and Delete, and use it in the file name field with dynamic content (`@dataset().fileName`).
4. **Build the pipeline:**
   - Add **Get Metadata** pointing at the source container, with the **Child Items** field selected.
   - Add **ForEach** after it, with Items set to `@activity('Get Metadata').output.childItems`.
   - Inside ForEach, add **Copy Data** (source file name = `@item().name`) followed on success by **Delete** (file name = `@item().name`).
5. **Create the trigger:** Add trigger → New/Edit → type **Storage events**, pick the storage account and the `triggersource` container, and choose the **Blob created** event.
6. **Publish** the changes. Triggers only start running after you publish and the trigger is activated.
7. **Test it** by uploading a file to `triggersource` and checking **Monitor → Pipeline runs**.

> **Note:** Storage event triggers rely on Azure Event Grid. If the trigger fails to save, make sure the `Microsoft.EventGrid` resource provider is registered on your subscription.

## Result

In the Monitor tab, a successful run shows all activities as `Succeeded`: one `Get Metadata`, one `ForEach`, and a `Copy Data` plus `DeleteCopiedFile` pair for every file that was in the source container.

## Screenshots

### ForEach items expression
![ForEach items expression](Screenshot1%20ForEach%20Items%20Expression.png)

### Source dataset parameter
![Source dataset parameter](Screenshot2%20Source%20Dataset%20Parameter.png)

### Copy activity parameter mapping
![Copy activity parameter mapping](Screenshot3%20Copy%20Activity%20Parameter%20Mapping.png)

### Successful pipeline run
![Successful pipeline run](Successful%20Pipeline%20Run.png)

### Successful trigger run
![Successful trigger run](Successful%20Trigger%20Run.png)
