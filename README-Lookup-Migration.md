
## Migration Pipeline Development Guide (Lookup Types)

This guide helps building migration pipelines in Azure Data Factory (ADF), using `pl-lookup-address-type` as a reference. It explains how to handle lookup types, insert missing entries, and populate the `Lookup` table using reference tables from the CMS2 database.


---

### 🔧 Prerequisites
Before starting, make sure the following are already set up:

1. **Self-hosted Integration Runtime** registered and connected to:
   - CMS2 database
   - Admin Portal database

2. **Dynamic datasets**:
   - `ds_cms2_tables` — parameter: `TableName`
   - `ds_adminportal_tables` — parameter: `TableName`

3. **Linked Services**:
   - `ls_cms2` (for CMS2 DB)
   - `ls_admin_portal` (for Admin Portal DB)

---

## ✅ Pipeline Template: `pl-lookup-address-type`

### Purpose
Migrates data from CMS2 table `tlkpAddressType` to Admin Portal's `Lookup` and `LookupType` tables.

---

## 📋 Step-by-Step Instructions

### 📌 Create a New Pipeline
1. Go to **Author** tab in ADF.
2. Right-click the **Pipelines** folder > **New Pipeline**.
3. Name it: `pl-lookup-address-type`

---

### 🎛️ Define Pipeline Variables
1. Click the **Variables** tab in the pipeline editor.
2. Add the following variables:

| Name              | Type   | Default Value                    |
|-------------------|--------|----------------------------------|
| vLookupTypeName   | String | `Address Type`                   |
| vLookupTypeDesc   | String | `Address Type Description`       |
| vLookupTypeId     | String | *(leave empty)*                  |

---

### 🧠 Step 1: Find Existing LookupType ID
1. Add a **Lookup** activity, name it `FindLookupTypeId`
2. Go to **Settings** tab:
   - Dataset: `ds_adminportal_tables`
   - TableName: `LookupType`
   - Query:
     ```sql
     SELECT Id FROM LookupType WHERE name = '@{variables('vLookupTypeName')}'
     ```
     > ⚠️ Click `Expression` (fx) button when pasting the query.

---

### 🧠 Step 2: Check if LookupType Exists
1. Add **If Condition**, name it: `If_LookupType_Exists`
2. Expression:
   ```adf
   @and(
       contains(activity('FindLookupTypeId').output, 'firstRow'),
       not(equals(activity('FindLookupTypeId').output.firstRow.Id, null))
   )
   ```

#### ✅ True Branch
- Add **Set Variable**: `Set vLookupTypeId`
- Value:
  ```adf
  @activity('FindLookupTypeId').output.firstRow.Id
  ```

#### ❌ False Branch
1. **Script activity**: `Insert Into LookupType`
   - Linked Service: `ls_admin_portal`
   - Script:
     ```sql
     INSERT INTO LookupType (Id, Name, Description, CreatedOn, LastModifiedOn, CreatedBy, LastModifiedBy)
     VALUES (NEWID(), '@{variables('vLookupTypeName')}', '@{variables('vLookupTypeDesc')}', GETUTCDATE(), GETUTCDATE(), '00000000-0000-0000-0000-000000000000', '00000000-0000-0000-0000-000000000000')
     ```

2. Add **Lookup**: `FindNewLookupTypeId`
   - Same query as earlier
3. **Set Variable**: `Set vLookupTypeId_new`
   - Value:
     ```adf
     @activity('FindNewLookupTypeId').output.firstRow.Id
     ```

---

### 🧠 Step 3: Load Source Address Types
1. Add a **Lookup** activity: `GetAddressType`
2. Settings:
   - Dataset: `ds_cms2_tables`
   - TableName: `tlkpAddressType`
   - First row only: `false`

---

### 🔁 Step 4: Insert into Lookup Table
1. Add a **ForEach** activity: `ForEachAddressType`
2. Items:
   ```adf
   @activity('GetAddressType').output.value
   ```
3. Inside ForEach:
   - Add **Script activity**: `InsertAddressType`
   - Linked Service: `ls_admin_portal`
   - Script:
     ```sql
     SET IDENTITY_INSERT Lookup ON;
     INSERT INTO Lookup (
         Id, Code, Name, Description, LookupTypeId, CreatedBy,
         CreatedOn, LastModifiedBy, LastModifiedOn
     )
     VALUES (
         NEWID(),
         @{item().AddressTypeId},
         '@{item().AddressType}',
         '@{item().AddressType}',
         '@{variables('vLookupTypeId')}',
         '00000000-0000-0000-0000-000000000000',
         '@{item().CreateDate}',
         '00000000-0000-0000-0000-000000000000',
         GETUTCDATE()
     )
     ```

---

### ✅ Step 5: Return Status
1. Add a **Set Variable**: `Return Success`
   - Depends on `ForEachAddressType` = `Succeeded`
   - Value:
     ```json
     [ { "key": "Status", "value": "Success" } ]
     ```

2. Add a **Set Variable**: `Return Error`
   - Depends on `ForEachAddressType` = `Failed`
   - Value:
     ```json
     [ { "key": "Status", "value": "Error" } ]
     ```

---

## 📦 Controller Pipeline: `pl-lookup-migration`

### Purpose
Orchestrates all lookup-type migration pipelines.

### Steps
1. Go to Author tab > Create new pipeline: `pl-lookup-migration`
2. Add multiple **Execute Pipeline** activities, one for each `pl-lookup-*` pipeline
3. Set all `Execute Pipeline` activities to run **in parallel**
4. Add a **Set Variable** at the end named `vReferenceMigrationComplete` with value `true`
5. Set `dependsOn` from all `Execute Pipeline` activities with condition `Succeeded`

---

### Included Pipelines
- `pl-lookup-filing-frequency`
- `pl-lookup-organization-group`
- `pl-lookup-title`
- `pl-lookup-phone-type`
- `pl-lookup-salutation-suffix`
- `pl-lookup-teaching-status`
- `pl-lookup-relationship-category`
- `pl-lookup-sign-on-type`
- `pl-lookup-address-type`
- `pl-lookup-salutation-prefix`
- `pl-lookup-organization-status`
- `pl-lookup-ems-region`
- `pl-lookup-contact-reason-category`
- `pl-lookup-lastname-suffix`

---

## 🧱 Naming Conventions
| Type               | Prefix     | Example                        |
|--------------------|------------|--------------------------------|
| Pipeline           | `pl-`      | `pl-lookup-address-type`       |
| Dataset            | `ds_`      | `ds_cms2_tables`, `ds_adminportal_tables` |
| Linked Service     | `ls_`      | `ls_admin_portal`              |
| Variable           | `v` prefix | `vLookupTypeName`              |

---

## 🧠 Developer Notes
- Each reference table should have its own `pl-lookup-<name>` pipeline.
- Every pipeline returns `pipelineReturnValue` as output.
- Controller pipeline can skip or retry failed lookups if needed.
- Use `@equals(firstRow, null)` to check existence safely.

