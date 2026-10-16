# Power Platform Resource Deletion and Solution Lifecycle Guide

## Purpose

This guide explains how to classify, delete, and retire Power Platform resources safely across Development, Test, UAT, and Production. It uses generic names such as **Solution A**, **App A**, **App B**, **Flow A**, and **Table A** so the guidance can be reused in any tenant.

Resources covered:

1. Canvas Apps
2. Cloud Flows
3. SharePoint customized forms
4. Model-driven Apps
5. Desktop Flows
6. Managed and unmanaged solutions
7. Power Platform CLI, PowerShell, Dataverse Web API, and UI approaches

> **Primary rule:** Identify the resource type, managed state, solution ownership, dependencies, environment, and solution layers before selecting a deletion method.

---

# 1. Solution concepts

## 1.1 What is a solution?

A solution is the Power Platform application lifecycle management container used to group and transport components such as:

- Canvas Apps
- Model-driven Apps
- Cloud Flows
- Desktop Flows
- Tables, columns, relationships, forms, and views
- Custom pages
- Connection references
- Environment variables
- Dashboards, charts, web resources, and other customizations

A component can be directly visible or nested under another component. For example, forms, views, columns, relationships, and business rules can be dependent on a table.

## 1.2 Unmanaged solution

An unmanaged solution is the development/source form of a solution.

Recommended use:

```text
Development environment
→ Create and edit Solution A as unmanaged
→ Commit the source to Git
→ Build or export a managed deployment package
```

Important behavior:

- Unmanaged solutions are intended for development.
- They can be exported as unmanaged or managed.
- The unmanaged solution should be considered the source for Power Platform assets.
- Deleting an unmanaged solution deletes only the solution container.
- The unmanaged customizations remain in the environment and become part of the default solution.

Therefore:

```text
Delete unmanaged Solution A
≠
Delete every App, Flow, Table, and customization previously listed in Solution A
```

## 1.3 Managed solution

A managed solution is the deployed form of the solution and is normally used in Test, UAT, SIT, and Production.

Recommended lifecycle:

```text
DEV: Solution A (unmanaged source)
          |
          +--> Git/source control
          |
          +--> Build/export as managed
                         |
                         +--> TEST: Solution A (managed)
                         |
                         +--> UAT: Solution A (managed)
                         |
                         +--> PROD: Solution A (managed)
```

Important behavior:

- Components can't be edited directly inside a managed solution.
- If managed properties allow customization, a managed component can be added to an unmanaged solution and customized.
- Such a customization creates an unmanaged layer/dependency over the managed component.
- A managed solution can't be exported as unmanaged.
- Uninstalling a managed solution removes its included customizations and extensions, subject to dependencies and solution layers.
- Uninstalling can remove data in custom tables or custom columns owned by the solution. Perform data-impact review first.

## 1.4 Solution lifecycle actions

| Action | Meaning | Deletes omitted components? | Recommended use |
|---|---|---:|---|
| **Create** | Author and export an unmanaged source solution | Not applicable | Initial development |
| **Update** | Deploy additions or modifications to an existing managed solution | **No** | Add or modify without deleting omitted components |
| **Upgrade** | Install a newer managed version and apply upgrade logic | **Yes** | Retire selected managed Apps, Flows, or other components |
| **Stage for Upgrade** | Install the newer version while temporarily retaining the previous version | After upgrade is applied | Data migration or controlled cutover |
| **Patch** | Layer a small set of changes over a parent managed solution | **No** | Small fixes, not component deletion |
| **Uninstall/Delete** | Remove the entire managed solution | Removes solution-owned components, subject to dependencies | Complete solution retirement |

### Example: retire App B and Flow B only

Current target:

```text
Solution A v1.0.0.0 (managed)
├── App A
├── App B
├── Flow A
├── Flow B
└── Table A
```

Desired target:

```text
Solution A v2.0.0.0 (managed)
├── App A
├── Flow A
└── Table A
```

Procedure:

1. Open the unmanaged source of Solution A in DEV.
2. Remove App B and Flow B after dependency review.
3. Increment the version.
4. Build/export Solution A as managed.
5. Deploy to Test/UAT using **Upgrade**.
6. Validate App A, Flow A, and Table A.
7. Promote the exact managed artifact to Production using **Upgrade**.

Do not use **Update**, because omitted components aren't deleted by Update. Do not use `pac solution delete`, because that targets the entire solution.

## 1.5 Publisher and publisher prefix

Every solution has a publisher. The publisher is associated with the ownership of components created in the solution and affects what other publishers are allowed to change.

A publisher also has a prefix that helps prevent naming collisions.

Example:

```text
Publisher: Organization Publisher
Prefix: org

Logical names:
org_tablea
org_columna
org_appa
```

Recommended practice:

- Create a meaningful organizational publisher.
- Use the same approved publisher consistently across environments.
- Avoid relying on the default publisher for production ALM.
- Choose the publisher before creating components because logical names can't be changed later.
- Once a component has been introduced through a managed solution, its publisher can't be changed.

## 1.6 Solution dependencies

Solutions and individual components can depend on other components.

Example:

```text
Base Solution
├── Table A
└── Column A

Extension Solution
└── App A uses Table A and Column A
```

The Extension Solution depends on the Base Solution. The Base Solution can't be uninstalled until the dependent solution or dependency is removed.

Always check dependencies before deleting:

- Apps
- Flows
- Tables and columns
- Forms and views
- Site maps
- Custom pages
- Connection references
- Environment variables
- Parent/child flows

## 1.7 Managed properties and unmanaged layers

Managed properties determine whether downstream customization is allowed.

This combination is possible:

```text
IsManaged = true
IsCustomizable = true
HasUnmanagedLayer = true
```

Meaning:

- The component has a managed base.
- Customization is permitted.
- A later unmanaged customization exists above the managed base.
- The active unmanaged layer can determine runtime behavior.

In UAT and Production, treat unmanaged layers as configuration drift unless approved.

To inspect manually:

```text
Power Apps
→ Solutions
→ Open the solution
→ Select the component
→ Advanced
→ See solution layers
```

---

# 2. How to understand the Resource Deletion Matrix

The matrix is a **decision guide**, not a sequence of commands.

## Step 1: Find the correct resource row

Start with the actual resource type:

- Classic Canvas App
- Component Library
- Custom Canvas Page
- Cloud Flow
- Desktop Flow
- SharePoint customized form
- Model-driven App
- Entire solution

Do not treat every record as an ordinary Canvas App or Cloud Flow.

## Step 2: Use the Identification column

This column shows which Dataverse table and property identify the resource.

```text
Canvas Apps    → canvasapps + canvasapptype
Cloud Flows    → workflows + category 5
Desktop Flows  → workflows + category 6
Model Apps     → appmodules
Solutions      → solutions
```

## Step 3: Determine managed state

Use `ismanaged` from Dataverse or the Managed column in the solution UI.

```text
ismanaged = false
→ Follow the Unmanaged deletion path

ismanaged = true
→ Follow the Managed deletion path
```

Being present in a solution does not automatically mean the resource is managed.

## Step 4: Decide the deletion scope

```text
Remove from solution
→ Resource remains in the environment

Delete from environment
→ Resource is deleted from the environment

Remove from managed package and Upgrade
→ Selected managed resource is retired while the solution remains

Delete/uninstall managed solution
→ Entire solution is retired
```

## Step 5: Select the technique

- **UI/OOB:** one-off validated cleanup.
- **PowerShell/API:** controlled automation for eligible unmanaged resources.
- **PAC CLI:** solution lifecycle operations, not universal App/Flow deletion.
- **Managed Upgrade:** selected managed component retirement.
- **Managed solution uninstall:** complete solution retirement.

## Step 6: Apply environment controls

| Environment | Recommended controls |
|---|---|
| DEV | Maker/admin validation and source-control update |
| Test | Automated validation and dependency checks |
| UAT | Approval, backup, and lower-environment evidence |
| PROD | Change approval, explicit environment targeting, data review, and post-validation |

## Step 7: Validate the result

1. Confirm the intended resource is absent.
2. Confirm retained Apps and Flows still work.
3. Confirm solution import/upgrade history succeeded.
4. Refresh CoE/governance inventory.
5. Record evidence, status, timestamp, and errors.

---

# 3. Resource Deletion Matrix

Read each row in this order:

```text
Identification
→ Managed state
→ Deletion scope
→ Supported method
→ Validation
```

| Resource | Identification | Unmanaged/standalone deletion | Managed deletion | Preferred automation |
|---|---|---|---|---|
| Classic Canvas App | `canvasapps`, `canvasapptype=0` | UI or `Remove-AdminPowerApp` | Remove from DEV source and deploy managed Upgrade | PowerShell + solution lifecycle |
| App Component Library | `canvasapps`, `canvasapptype=1` | Check consuming apps, then delete | Remove from source and Upgrade | Dataverse inventory + Upgrade |
| Custom Canvas Page | `canvasapps`, `canvasapptype=2` | Remove Model-driven App references first | Remove from source and Upgrade | Solution lifecycle |
| Unified App | `canvasapps`, `canvasapptype=3` | Validate dependencies | Remove from source and Upgrade | Solution lifecycle |
| Code App | `canvasapps`, `canvasapptype=4` | Validate app-specific dependencies | Remove from source and Upgrade | Solution lifecycle |
| Mobile App | `canvasapps`, `canvasapptype=5` | Validate dependencies | Remove from source and Upgrade | Solution lifecycle |
| Cloud Flow | `workflows`, `category=5` | Remove from solution or delete from environment | Remove from DEV source and Upgrade | Dataverse/API + lifecycle |
| Desktop Flow | `workflows`, `category=6` | Delete from Desktop Flows after checks | Remove through managed solution lifecycle | UI/API + lifecycle |
| SharePoint customized form | Canvas-based app associated with a SharePoint list and `SharePointIntegration` | Restore standard list form, validate, then assess orphaned app | Evaluate underlying solution state | SharePoint + Power Apps process |
| Model-driven App | `appmodules` | Show Dependencies, delete app, evaluate site map | Remove from DEV source and Upgrade | Solution lifecycle/API |
| Entire unmanaged solution | `solutions`, `ismanaged=false` | Deletes only solution container | Not applicable | UI/PAC with caution |
| Entire managed solution | `solutions`, `ismanaged=true` | Not applicable | Uninstall whole solution | `pac solution delete` |
| Selected managed Apps/Flows | Owning solution + component IDs | Not applicable | Remove from source and deploy Upgrade | PAC import/upgrade pipeline |

---

# 4. Canvas Apps

## 4.1 Canvas App type values

| `canvasapptype` | Meaning |
|---:|---|
| 0 | Classic Canvas App |
| 1 | App Component Library |
| 2 | Custom Canvas Page |
| 3 | Unified App |
| 4 | Code App |
| 5 | Mobile App |

## 4.2 Retrieve details

```http
GET https://<org>.crm.dynamics.com/api/data/v9.2/canvasapps(<APP-ID>)?$select=canvasappid,displayname,name,canvasapptype,canvasappsubtype,ismanaged,solutionid,componentstate,status,createdtime,lastmodifiedtime
```

Generic example:

```json
{
  "displayname": "App A",
  "canvasappid": "11111111-1111-1111-1111-111111111111",
  "canvasapptype": 0,
  "ismanaged": false,
  "solutionid": "22222222-2222-2222-2222-222222222222",
  "componentstate": 0,
  "status": "Ready"
}
```

Interpretation:

```text
canvasapptype = 0  → Classic Canvas App
ismanaged = false → Unmanaged component
componentstate = 0 → Published
```

## 4.3 Delete unmanaged Canvas App from UI

```text
Power Apps
→ Correct environment
→ Apps
→ Select App A
→ Delete
→ Delete from cloud
```

## 4.4 Delete unmanaged Canvas App with PowerShell

```powershell
$EnvironmentId = "<ENVIRONMENT-ID>"
$AppId = "<APP-ID>"

$app = Get-AdminPowerApp `
    -EnvironmentName $EnvironmentId `
    -AppName $AppId

$app | Format-List AppName, DisplayName, EnvironmentName, Owner

Remove-AdminPowerApp `
    -EnvironmentName $EnvironmentId `
    -AppName $AppId
```

## 4.5 Delete managed App B but retain Solution A

```text
DEV
→ Remove App B from unmanaged source Solution A
→ Check dependencies
→ Increment version
→ Build managed Solution A
→ UAT Upgrade
→ Validate App A and Flow A
→ PROD Upgrade
```

Do not use `pac solution delete` because the complete Solution A must remain.

---

# 5. Cloud Flows

## 5.1 Identify Cloud Flow and Workflow ID

```http
GET https://<org>.crm.dynamics.com/api/data/v9.2/workflows?$select=workflowid,name,category,ismanaged,solutionid,statecode,statuscode&$filter=name eq 'Flow A'
```

Categories:

```text
category = 5 → Cloud Flow
category = 6 → Desktop Flow
```

PowerShell example:

```powershell
$EnvironmentUrl = "https://<org>.crm.dynamics.com/"
$FlowName = "Flow A"
$escapedFlowName = $FlowName.Replace("'", "''")

$uri = "${EnvironmentUrl}api/data/v9.2/workflows?`$select=workflowid,name,category,ismanaged,solutionid,statecode,statuscode&`$filter=name eq '$escapedFlowName'"

$result = Invoke-RestMethod `
    -Method Get `
    -Uri $uri `
    -Headers $headers

$result.value | Format-Table workflowid,name,category,ismanaged,statecode,statuscode
```

## 5.2 Remove from solution versus delete

```text
Remove from this solution
→ Flow remains in the environment
→ Flow becomes available through the Default Solution

Delete from this environment
→ Flow is deleted from the environment
```

UI:

```text
Power Automate
→ Solutions
→ Open Solution A
→ Select Flow B
→ Commands / Remove
→ Choose required action
```

## 5.3 Retire managed Flow B

```text
Solution A v1
├── App A
├── Flow A
└── Flow B
```

Procedure:

1. Remove Flow B from the unmanaged DEV source.
2. Update related Apps, child flows, connection references, and environment variables.
3. Increment the version.
4. Build/export managed.
5. Deploy to UAT using Upgrade.
6. Validate Flow A and App A.
7. Deploy to Production using Upgrade.

---

# 6. SharePoint Customized Forms

## 6.1 Why separate treatment is needed

A customized SharePoint list form is a Canvas-based app integrated with a specific SharePoint list through `SharePointIntegration`.

Before cleanup, capture:

```text
Environment ID
App ID
App display name
SharePoint site URL
SharePoint list name
SharePoint list ID
Owner
Last modified date
Managed state
Solution ID
```

## 6.2 Recommended sequence

```text
Identify owning SharePoint list
→ Confirm customized form is obsolete
→ Restore standard/default list form
→ Validate New, View, and Edit operations
→ Check whether Canvas App record is orphaned
→ Unmanaged: direct eligible cleanup
→ Managed: retire through solution Upgrade
```

Don't delete the Canvas App first without understanding the SharePoint list association.

---

# 7. Model-driven Apps

## 7.1 Retrieve details

```http
GET https://<org>.crm.dynamics.com/api/data/v9.2/appmodules(<APP-MODULE-ID>)?$select=appmoduleid,name,uniquename,ismanaged,solutionid,componentstate,createdon,modifiedon
```

## 7.2 Delete unmanaged Model-driven App

```text
Power Apps
→ Solutions
→ Open source solution
→ Select App A
→ Show Dependencies
→ Resolve blockers
→ Delete
→ Delete from this environment
→ Evaluate associated site map cleanup
```

## 7.3 Retire managed Model-driven App

```text
DEV source Solution A
→ Remove App B
→ Resolve site map/table/form/view/custom-page dependencies
→ Increment version
→ Build managed
→ UAT Upgrade
→ Validate retained components
→ PROD Upgrade
```

---

# 8. Desktop Flows

## 8.1 Identify

```http
GET https://<org>.crm.dynamics.com/api/data/v9.2/workflows?$select=workflowid,name,category,ismanaged,solutionid,statecode,statuscode&$filter=category eq 6
```

## 8.2 UI deletion

```text
Power Automate
→ My flows
→ Desktop flows
→ Select Desktop Flow A
→ Delete
```

## 8.3 Dependency checks

Validate:

- Calling Cloud Flows
- Child Desktop Flows
- Machine and machine-group associations
- Connections and credentials
- Schedules
- Owners and shared users
- Solution membership
- Managed state

For a managed Desktop Flow, remove it from the DEV source and deploy a managed Upgrade.

---

# 9. PAC CLI Lifecycle

## 9.1 Always verify the environment

```powershell
pac auth list
pac solution list --environment "https://<target-org>.crm.dynamics.com"
```

If `--environment` is omitted, PAC uses the Dataverse organization selected in the active authentication profile.

## 9.2 Delete complete managed Solution A

```powershell
pac solution delete `
    --solution-name SolutionA `
    --environment "https://<target-org>.crm.dynamics.com"
```

Use only when the entire Solution A is retired.

## 9.3 Import managed artifact

```powershell
pac solution import `
    --path ".\artifacts\SolutionA_managed.zip" `
    --environment "https://<uat-org>.crm.dynamics.com"
```

Confirm options supported by the installed PAC version:

```powershell
pac solution import --help
pac solution upgrade --help
```

## 9.4 Selected managed component retirement pipeline

```text
DEV unmanaged Solution A
→ Remove App B / Flow B
→ Commit source changes
→ Increment version
→ Build managed artifact
→ Import/Upgrade Test
→ Validate
→ Import/Upgrade UAT
→ Approve
→ Import/Upgrade PROD
→ Refresh inventory
```

---

# 10. Dataverse API Authentication

## 10.1 PowerShell 7.4+

```powershell
$EnvironmentUrl = "https://<org>.crm.dynamics.com/"

Connect-AzAccount

$secureToken = (
    Get-AzAccessToken `
        -ResourceUrl $EnvironmentUrl `
        -AsSecureString
).Token

$token = ConvertFrom-SecureString `
    -SecureString $secureToken `
    -AsPlainText

$headers = @{
    Authorization       = "Bearer $token"
    Accept              = "application/json"
    "OData-MaxVersion" = "4.0"
    "OData-Version"    = "4.0"
}

Invoke-RestMethod `
    -Uri "${EnvironmentUrl}api/data/v9.2/WhoAmI" `
    -Method Get `
    -Headers $headers
```

## 10.2 Windows PowerShell 5.1

```powershell
$secureToken = (
    Get-AzAccessToken -ResourceUrl $EnvironmentUrl
).Token

$ptr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($secureToken)

try {
    $token = [Runtime.InteropServices.Marshal]::PtrToStringBSTR($ptr)
}
finally {
    [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($ptr)
}

$headers = @{
    Authorization       = "Bearer $token"
    Accept              = "application/json"
    "OData-MaxVersion" = "4.0"
    "OData-Version"    = "4.0"
}
```

Validate `WhoAmI` before calling App or Flow APIs.

---

# 11. Pre-deletion Checklist

| Check | Evidence |
|---|---|
| Correct environment | Environment ID and Dataverse URL |
| Correct resource | Display name, unique name, and GUID |
| Resource subtype | `canvasapptype` or workflow `category` |
| Managed state | `ismanaged` |
| Owning solution | Solution ID and unique name |
| Solution type | Managed or unmanaged |
| Unmanaged layer | Solution Layers review |
| Dependencies | Dependency result or UI evidence |
| Usage | Last activity/run and owner validation |
| Ownership | Owner and co-owners |
| Data impact | Tables, columns, and data affected |
| Approval | Approver and change/ticket reference |
| Recovery | Backup/export/source-control version |
| Execution method | UI, PowerShell, PAC, API, or Upgrade |
| Post-validation | Resource absent and retained workloads tested |

---

# 12. Suggested Governance Inventory

```text
ResourceType
ResourceSubtype
DisplayName
ResourceId
EnvironmentId
EnvironmentName
EnvironmentType
Owner
IsManaged
IsCustomizable
HasUnmanagedLayer
SolutionId
SolutionUniqueName
SolutionIsManaged
Publisher
DependencyCount
LastActivityDate
CleanupEligibility
CleanupMethod
ApprovalStatus
ApprovedBy
DeletionStatus
DeletedOn
ValidationStatus
ErrorMessage
```

Suggested statuses:

```text
Discovered
Classified
Pending Owner Review
Pending Dependency Review
Approved
Excluded
Scheduled for Cleanup
Cleanup Running
Deleted
Failed
Post-Validation Complete
```

---

# 13. End-to-End Generic Examples

## Example A: standalone unmanaged App A

```text
Resource: App A
Type: Classic Canvas App
ismanaged: false
Usage: inactive
Approval: approved
```

```powershell
Remove-AdminPowerApp `
    -EnvironmentName "<ENVIRONMENT-ID>" `
    -AppName "<APP-ID>"
```

Then verify absence and refresh inventory.

## Example B: retire managed App B only

```text
Solution A managed in UAT/PROD
App A retained
App B retired
```

```text
DEV: remove App B from unmanaged source Solution A
→ version Solution A
→ build managed artifact
→ UAT Upgrade
→ validate App A
→ PROD Upgrade
```

## Example C: remove unmanaged Flow B from solution only

```text
Power Automate
→ Solution A
→ Flow B
→ Remove
→ Remove from this solution
```

Flow B remains in the environment.

## Example D: delete unmanaged Flow B from environment

```text
Power Automate
→ Solution A
→ Flow B
→ Remove
→ Delete from this environment
```

Flow B is removed from the environment.

## Example E: retire complete managed Solution A

```powershell
pac solution list `
    --environment "https://<target-org>.crm.dynamics.com"

pac solution delete `
    --solution-name SolutionA `
    --environment "https://<target-org>.crm.dynamics.com"
```

Use only after dependency and data-impact review.

---

# 14. Microsoft Learn References

- [Solution concepts with Power Platform](https://learn.microsoft.com/en-us/power-platform/alm/solution-concepts-alm)
- [Delete a Canvas App](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/delete-app)
- [Remove-AdminPowerApp](https://learn.microsoft.com/en-us/powershell/module/microsoft.powerapps.administration.powershell/remove-adminpowerapp?view=pa-ps-latest)
- [Canvas App Web API entity](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/webapi/reference/canvasapp?view=dataverse-latest)
- [Remove solution-aware Cloud Flows](https://learn.microsoft.com/en-us/power-automate/remove-solution-aware-flow)
- [Manage Cloud Flows with code](https://learn.microsoft.com/en-us/power-automate/manage-flows-with-code)
- [Delete a Model-driven App](https://learn.microsoft.com/en-us/power-apps/maker/model-driven-apps/delete-model-driven-app)
- [Model-driven App Web API entity](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/webapi/reference/appmodule?view=dataverse-latest)
- [Manage Desktop Flows](https://learn.microsoft.com/en-us/power-automate/desktop-flows/manage)
- [SharePoint forms integration](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/sharepoint-form-integration)
- [Update or upgrade a solution](https://learn.microsoft.com/en-us/power-apps/maker/data-platform/update-solutions)
- [Solution layers](https://learn.microsoft.com/en-us/power-apps/maker/data-platform/solution-layers)
- [Removing dependencies](https://learn.microsoft.com/en-us/power-platform/alm/removing-dependencies)
- [PAC CLI solution commands](https://learn.microsoft.com/en-us/power-platform/developer/cli/reference/solution)
- [Dataverse Web API authentication](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/webapi/authenticate-web-api)

---

# Final Rule

```text
Unmanaged resource
→ Direct supported deletion after dependency and approval checks

Managed individual App or Flow
→ Remove from DEV source and deploy a managed Upgrade

Entire managed solution
→ Uninstall/delete after dependency and data-impact review

UAT/PROD unmanaged layer
→ Treat as configuration drift and review before cleanup
```
