# 🛠️ Steps to Set Up and Use the Database Object Deployment Pipeline

This process outlines how to create, modify, and deploy SQL database objects such as stored procedures, views, and synonyms using an Azure DevOps pipeline.

> ✅ **Note**: This pipeline currently supports any `.sql`-based object that can be diffed and executed through text (e.g., stored procedures, views, synonyms).

---

## 📁 1. Clone the Repository

- Open **VS Code** or your preferred Git client.
- Clone the target repository from Azure DevOps.

---

## 🌿 2. Create a Feature Branch

```bash
git checkout -b feature/<your-feature-name>
```
- Or user VS Code or VS 2022 to commit changes
- Follow naming convention for feature branches, e.g., `feature/<username>-<branch-name>`.

---

## 🚀 3. Make Changes

- Navigate to the `StoredProcedures`, `Views`, or `Synonyms` folder.
- Modify or add the necessary `.sql` files.
- Ensure the file contains `CREATE OR ALTER` (for SQL Server 2016 SP1+).
  - For older versions, use:

```sql
IF OBJECT_ID('dbo.MyProc', 'P') IS NULL
    EXEC('CREATE PROCEDURE dbo.MyProc AS BEGIN SET NOCOUNT ON; END');
GO

ALTER PROCEDURE dbo.MyProc
AS
BEGIN
    -- procedure logic
END;
```

---

## 🔄 4. Publish Your Branch

Push the branch to remote:

```bash
git push --set-upstream origin feature/<your-feature-name>
```
- Or user VS Code or VS 2022 to publish branch
---

## ✅ 5. Verify the Pipeline Setup

- Confirm that `azure-pipeline.yml` and any referenced templates (e.g., `deploy-sql-template.yml`) exist in the repo.
- No pipeline should run yet unless you're already on a `main` or `release/*` branch.

---

## 💬 6. Commit and Create Pull Request

- Commit your changes with a clear message.
- Push your commits.
- Create a **Pull Request** into `main`.
- Enable **Auto-complete** to automatically merge after approvals.

---

## 🔁 7. Merge into `main` (Triggers DEV Deployment)

- Once merged to `main`, the pipeline:
  - Detects modified `.sql` files
  - Validates syntax using `sqlcmd`
  - Deploys changes to the **DEV** environment

---

## 🧪 8. Validate the Changes

- Review the pipeline run in Azure DevOps.
- Ensure the `Deploy to DEV` stage completes successfully.
- Manually verify the updated objects in the **DEV database**.

---

## 🚢 9. Promote to Upper Environments

- To deploy to QA or PROD:

```bash
git checkout -b release/<release-name>
git push --set-upstream origin release/<release-name>
```
- This can be done in Azure DevOps UI as well.
- This triggers the pipeline for:
  - **Deploy to QA** (with optional approval)
  - **Deploy to PROD** (with required approval)

---

## 📌 Notes

- Use `CREATE OR ALTER` when supported, or safe `IF EXISTS` fallback otherwise.
- Avoid hardcoding environment-specific logic in SQL; use parameters or external configuration.
- Use Azure DevOps pipeline history for audit trails and traceability.
