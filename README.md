# 🚀 Azure DevOps Pipeline Rewiring with GitHub Repositories

This repository provides a GitHub Actions workflow for rewiring Azure DevOps pipelines so that their source repositories point to migrated GitHub repositories through an Azure DevOps GitHub service connection.

The workflow validates the required inputs, prepares the CSV files, installs the required GitHub CLI extension, executes the rewiring script, and publishes the generated logs as a workflow artifact.

---

## 📋 Workflow Execution Details

The GitHub Actions workflow contains two jobs. The first job validates the prerequisites, and the second job prepares the inputs and performs pipeline rewiring.

### Job 1️⃣: Prerequisite Validation

The `prerequisite-validation` job performs the following checks:

- Verifies that the `GH_PAT` GitHub Actions secret is configured.
- Verifies that the `ADO_PAT` GitHub Actions secret is configured.
- Verifies that `bash/pipelines.csv` exists and contains at least one data row.
- Checks whether `bash/repos_with_status.csv` exists and contains migration results.
- Verifies that `bash/4_rewire_pipeline.sh` exists.
- Normalizes Windows-style line endings in the CSV files.

> [!NOTE]
> If `bash/repos_with_status.csv` is missing or contains only a header, the workflow uses the fallback path and queues all entries from `bash/pipelines.csv`.

### Job 2️⃣: Pipeline Rewiring

The `pipeline-rewiring` job performs the following actions:

- Reads the required columns dynamically from `bash/pipelines.csv`.
- Filters pipelines based on repositories marked `Success` in `bash/repos_with_status.csv`.
- Generates the normalized `pipelines.csv` input required by the rewiring script.
- Generates a temporary `repos_with_status.csv` for the selected pipelines.
- Verifies that GitHub CLI is available on the runner.
- Installs the `github/gh-ado2gh` GitHub CLI extension.
- Fixes line endings and executable permissions for `bash/4_rewire_pipeline.sh`.
- Executes `bash/4_rewire_pipeline.sh`.
- Uploads generated rewiring logs as the `rewiring-logs` workflow artifact.
- Removes the generated status file after execution.

The rewiring script runs the following command for each selected pipeline:

```bash
gh ado2gh rewire-pipeline \
  --ado-org "$ADO_ORG" \
  --ado-team-project "$ADO_PROJECT" \
  --ado-pipeline "$ADO_PIPELINE" \
  --github-org "$GITHUB_ORG" \
  --github-repo "$GITHUB_REPO" \
  --service-connection-id "$SERVICE_CONNECTION_ID"
```

---

## 📁 Required Repository Structure

```text
.
├── .github
│   └── workflows
│       └── pipeline-rewiring.yml
├── bash
│   ├── 4_rewire_pipeline.sh
│   ├── pipelines.csv
│   └── repos_with_status.csv
└── README.md
```

`repos_with_status.csv` is optional for the workflow because fallback processing is supported. However, keeping it is recommended when rewiring must be limited to repositories that migrated successfully.

---

## 📄 CSV Configuration Files

Place the CSV input files in the `bash/` directory.

### `bash/pipelines.csv`

This file defines the Azure DevOps pipelines and their target GitHub repositories.

| Column | Description |
|---|---|
| `org` | Azure DevOps organization name. |
| `teamproject` | Azure DevOps project name. |
| `repo` | Azure DevOps repository name. This value is matched against `repos_with_status.csv`. |
| `pipeline` | Azure DevOps pipeline name or path, for example `\my-pipeline-ci`. |
| `url` | Azure DevOps pipeline URL for reference. The rewiring script does not require this column. |
| `serviceConnection` | Azure DevOps GitHub service connection ID in GUID format. |
| `github_org` | Target GitHub organization name. |
| `github_repo` | Target GitHub repository name. |

#### Example `pipelines.csv`

```csv
org,teamproject,repo,pipeline,url,serviceConnection,github_org,github_repo
v-biradarm,TestRepo2,fixpipeline1,\fixpipeline1,https://dev.azure.com/v-biradarm/TestRepo2/_build?definitionId=39,9bc1d24-177c-4cdc-9457-45caa069c04d,ADO-migration-test,fixpipeline11
v-biradarm,TestRepo2,fixpipeline2,\fixpipeline2,https://dev.azure.com/v-biradarm/TestRepo2/_build?definitionId=44,99bc1d24-177c-4cdc-9457-45caa069c04d,ADO-migration-test,fixpipeline21
```

> [!IMPORTANT]
> Replace all sample values with the correct Azure DevOps organization, project, repository, pipeline, service connection ID, GitHub organization, and GitHub repository values for your environment.

### `bash/repos_with_status.csv`

This file contains repository migration results and controls which repositories are eligible for pipeline rewiring.

- You can use the `repos_with_status.csv` file generated during the repository migration phase.
- When the file exists and contains data, it takes priority over fallback processing.
- Only repositories with the exact status `Success` are selected for rewiring.
- The `repo` value must match the corresponding `repo` value in `bash/pipelines.csv`.

Expected header:

```csv
org,teamproject,repo,github_org,github_repo,visibility,status
```

Example:

```csv
org,teamproject,repo,github_org,github_repo,visibility,status
v-biradarm,TestRepo2,fixpipeline1,ADO-migration-test,fixpipeline11,private,Success
v-biradarm,TestRepo2,fixpipeline2,ADO-migration-test,fixpipeline21,private,Success
```

---

## 🔐 Authentication Secrets

Create the following GitHub Actions secrets at either repository or organization level:

| Secret | Description |
|---|---|
| `GH_PAT` | GitHub personal access token used by GitHub CLI and the `gh-ado2gh` extension. |
| `ADO_PAT` | Azure DevOps personal access token used to access and update Azure DevOps pipelines. |

### Configure Repository Secrets

1. Open the GitHub repository.
2. Select **Settings**.
3. Select **Secrets and variables**.
4. Select **Actions**.
5. Select **New repository secret**.
6. Create `GH_PAT`.
7. Create `ADO_PAT`.

> [!CAUTION]
> Never commit PAT values directly to the workflow, shell script, CSV files, or repository history.

### GitHub PAT Permissions

Configure the GitHub token with the permissions required by your organization and migration process. Common migration-related classic PAT scopes include:

- `repo`
- `workflow`
- `admin:org`
- `read:user`

Always follow your organization's least-privilege and credential-management requirements.

### Azure DevOps PAT Permissions

The Azure DevOps token must be able to read pipeline information and perform the required rewiring operation. Depending on your organization's configuration, relevant areas can include:

- Build
- Code
- GitHub Connections
- Graph
- Identity
- Pipeline Resources
- Project and Team
- Security
- Service Connections
- User Profile

Use the minimum permissions that successfully support your validated rewiring process. Avoid using full access unless it is explicitly approved for your environment.

---

## 🔗 Azure DevOps GitHub Service Connection

Each `pipelines.csv` row must include the Azure DevOps GitHub service connection ID in the `serviceConnection` column.

To locate the service connection:

1. Open the Azure DevOps project.
2. Select **Project settings**.
3. Select **Service connections**.
4. Open the GitHub service connection used for the migrated repositories.
5. Copy the service connection ID.
6. Add the ID to the corresponding row in `bash/pipelines.csv`.

The value must be a real service connection ID. Empty values and placeholders such as `TODO`, `TBD`, `placeholder`, `xxx`, and `your-service-connection-id` are rejected by the rewiring script.

---

## 🛠️ Step-by-Step Instructions

### 1️⃣ Clone the Repository

```bash
git clone YOUR_REPOSITORY_URL
cd pipeline-rewiring-only
```

Replace `YOUR_REPOSITORY_URL` with your repository clone URL.

### 2️⃣ Add the GitHub Actions Workflow

Place the workflow file at:

```text
.github/workflows/pipeline-rewiring.yml
```

### 3️⃣ Prepare the CSV Files

Edit the pipeline configuration:

```bash
code bash/pipelines.csv
```

If repository migration status is available, add or update:

```bash
code bash/repos_with_status.csv
```

Before running the workflow, verify:

- Required headers are present.
- Repository names match between both CSV files.
- Target GitHub repositories exist.
- Service connection IDs are valid.
- Repositories intended for rewiring have the status `Success`.

### 4️⃣ Add GitHub Actions Secrets

Add the following secrets under **Settings → Secrets and variables → Actions**:

```text
GH_PAT
ADO_PAT
```

### 5️⃣ Commit and Push the Files

```bash
git add .github/workflows/pipeline-rewiring.yml bash/4_rewire_pipeline.sh bash/pipelines.csv README.md
git add bash/repos_with_status.csv
git commit -m "Add GitHub Actions pipeline rewiring workflow"
git push
```

If `repos_with_status.csv` is intentionally not included, omit its `git add` command.

### 6️⃣ Run the Workflow

1. Open the repository in GitHub.
2. Select the **Actions** tab.
3. Select **ADO Pipeline Rewiring**.
4. Select **Run workflow**.
5. Select the required branch.
6. Select **Run workflow** again.

---

## 📊 Monitor Workflow Execution

| Job | Key Actions |
|---|---|
| **Prerequisite Validation** | Validates secrets, required files, CSV content, and the rewiring script. |
| **Pipeline Rewiring** | Filters CSV inputs, installs `gh-ado2gh`, executes rewiring, and uploads logs. |

To inspect the results:

1. Open the workflow run.
2. Review the logs for both jobs.
3. Expand the **Execute pipeline rewiring** step.
4. Review the rewiring summary.
5. Download the `rewiring-logs` artifact from the workflow run summary.

---

## ✅ Verify Rewiring Success

After the workflow completes:

1. Open the relevant pipeline in Azure DevOps.
2. Edit or inspect the pipeline definition.
3. Confirm that the repository source points to the expected GitHub organization and repository.
4. Confirm that the expected GitHub service connection is selected.
5. Queue a controlled validation run if permitted by your environment.
6. Verify that the pipeline can check out the target GitHub repository.

---

## 📄 Rewiring Logs

The script generates a timestamped log file similar to:

```text
pipeline-rewiring-YYYYMMDD-HHMMSS.txt
```

The workflow uploads generated logs under the artifact name:

```text
rewiring-logs
```

The log contains:

- Total pipelines processed
- Successful rewiring count
- Pipelines already pointing to GitHub
- Pipelines skipped because rewiring was incomplete
- Failed pipeline count
- Per-pipeline results
- Detailed command output

---

## ⚠️ Important Behavior

- The workflow is manual and runs through `workflow_dispatch`.
- The workflow does not run automatically for pushes or pull requests.
- When `repos_with_status.csv` contains data, only repositories marked `Success` are queued.
- When the status file is missing or empty, all rows in `pipelines.csv` are queued through fallback processing.
- The workflow rewrites the checked-out `bash/pipelines.csv` during the run, but it does not commit the generated changes back to the repository.
- The generated `bash/repos_with_status.csv` is removed at the end of the workflow run.
- Rewiring warnings or partial failures may be reported by the script without terminating the entire workflow as a hard failure.

---

## 🔎 Troubleshooting

### Missing Secret

Example:

```text
Missing repository or organization secret: GH_PAT
```

Resolution:

- Confirm that both `GH_PAT` and `ADO_PAT` exist under GitHub Actions secrets.
- Confirm that the workflow is running in a context where those secrets are available.

### Missing CSV File

Example:

```text
Required file 'bash/pipelines.csv' was not found.
```

Resolution:

- Confirm the file is committed under the `bash/` directory.
- Confirm the capitalization and filename match exactly.

### Invalid Service Connection

Example:

```text
Invalid service connection IDs found
```

Resolution:

- Replace empty or placeholder values in the `serviceConnection` column.
- Verify that each value is the correct Azure DevOps GitHub service connection ID.

### Repository Not Selected for Rewiring

Resolution:

- Confirm the repository name matches in both CSV files.
- Confirm the repository status is exactly `Success`.
- Review the CSV filtering messages in the workflow logs.

### Script Permission or Line-Ending Issue

The workflow automatically executes:

```bash
sed -i 's/\r$//' bash/4_rewire_pipeline.sh
chmod +x bash/4_rewire_pipeline.sh
```

If the issue continues, confirm that the shell script is valid Bash and does not contain unsupported characters or malformed commands.

---

## 🧾 Summary

This workflow provides a controlled, manually triggered process to:

1. Validate required secrets and files.
2. Select pipelines associated with successfully migrated repositories.
3. Prepare normalized CSV inputs.
4. Install the `gh-ado2gh` extension.
5. Rewire Azure DevOps pipelines to GitHub repositories.
6. Publish detailed rewiring logs for verification and troubleshooting.
