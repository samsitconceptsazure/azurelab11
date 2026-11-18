# LAB 11 Project 16 — Student account setup (standard Azure subscription)

## 0. Prerequisites and cost
- Standard Azure subscription with permission to create resource groups, App Service plans, App Services, and Application Insights in **Central India**.
- Permission to register an Entra application and grant it Contributor on a lab resource group. If you cannot assign roles, ask your subscription administrator.
- GitHub account, GitHub Actions enabled, Azure CLI / Azure Cloud Shell, and optional Node.js 20 for local tests.
- **Costs:** B1 Linux App Service is billable while provisioned; Application Insights may incur ingestion charges. Create only DEV first; delete resources after lab. QA and PROD create additional billable plans.

## 1. Fork and locate workflow
1. Fork the instructor's `azure-cloud-arch-devops-ai` repository into **your own GitHub account**.
2. At the ROOT of the fork, create `.github/workflows/azure-deploy.yml` by copying `LAB 11 — ARM & Bicep Infrastructure as Code/LAB11-Project16-Azure-Application-Deployment-Pipeline/github-workflow/azure-deploy.yml`.
3. If the fork has the instructor's root workflow already, edit it rather than creating duplicates. GitHub Actions discovers workflows only at repository root.
4. If your fork uses a different project path, update `defaults.run.working-directory` in the workflow. **Do not** put workflow YAML only in the lab subdirectory.

## 2. Azure Cloud Shell — choose subscription and create resource groups
```bash
az login  # skip if already signed in to Azure Cloud Shell
az account list -o table
az account set --subscription '<YOUR_SUBSCRIPTION_ID>'
az account show --query '{subscription:id,tenant:tenantId}' -o table
az group create -n rg-lab11-p16-dev -l centralindia
```
For QA/PROD (optional), create `rg-lab11-p16-qa` and `rg-lab11-p16-prod` too.

## 3. Register YOUR OWN Entra application
Azure Portal > Microsoft Entra ID > App registrations > New registration. Name it `LAB11-Project16-Student-CICD`, single tenant. Copy **Application (client) ID** and **Directory (tenant) ID**. Then create its service principal if needed:
```bash
APP_ID='<YOUR_APPLICATION_CLIENT_ID>'
az ad sp create --id "$APP_ID"
```
If the service principal already exists, proceed. The app registration is **not** a GitHub OAuth app; it is an Azure workload identity.

## 4. Grant least-scope Azure RBAC
Azure Portal > Resource groups > `rg-lab11-p16-dev` > Access control (IAM) > Add role assignment > Contributor > Members: select your app's **service principal** > Review + assign.
CLI alternative, if you have role-assignment rights:
```bash
SUB_ID=$(az account show --query id -o tsv)
SP_ID=$(az ad sp show --id "$APP_ID" --query id -o tsv)
az role assignment create --assignee-object-id "$SP_ID" --assignee-principal-type ServicePrincipal --role Contributor --scope "/subscriptions/$SUB_ID/resourceGroups/rg-lab11-p16-dev"
```
Contributor on the resource group is enough for the lab deployment; it does not grant permission to create role assignments. Repeat only for QA/PROD groups if used.

## 5. Create GitHub environment and OIDC federated credential
GitHub fork > Settings > Environments > New environment > `dev`.
Microsoft Entra > App registrations > your app > Certificates & secrets > Federated credentials > Add credential > **GitHub Actions deploying Azure resources**.
- Organization: **YOUR** GitHub owner (personal username or organization)
- Organization ID: numeric ID for YOUR owner
- Repository: **YOUR** fork repository name
- Repository ID: numeric ID for YOUR fork
- Entity type: Environment
- Environment name: `dev`
- Audience: `api://AzureADTokenExchange`
Find numeric IDs using:
```bash
OWNER='<YOUR_GITHUB_OWNER>'
REPO='<YOUR_FORK_REPOSITORY_NAME>'
curl -fsSL "https://api.github.com/repos/$OWNER/$REPO" | python3 -c "import json,sys; d=json.load(sys.stdin); print('Owner ID:',d['owner']['id']); print('Repo ID:',d['id'])"
```
Check the **exact subject** generated in Entra against the `subject claim` in a GitHub Actions Azure Login log. Current Entra UI may generate immutable-ID format (`repo:OWNER@OWNER_ID/REPO@REPO_ID:environment:dev`). Do not copy the instructor's subject or IDs. If the account's Entra UI uses a different subject format, match the actual GitHub OIDC claim.

## 6. Add GitHub Actions secrets
In YOUR fork > Settings > Secrets and variables > Actions > New repository secret:
- `AZURE_CLIENT_ID` = YOUR Entra app's Application (client) ID
- `AZURE_TENANT_ID` = YOUR Directory (tenant) ID
- `AZURE_SUBSCRIPTION_ID` = YOUR subscription ID
These are identifiers, not client passwords. Never commit secrets or tokens to Git.

## 7. Run and verify
GitHub > Actions > LAB11 Project16 Azure Deployment > Run workflow > branch main > environment `dev`.
Expected stages: checkout, permissions, Azure OIDC login, Bicep validate, infrastructure deploy, ZIP application deploy, verify. Use Azure Portal > `rg-lab11-p16-dev` > App Service and open its `/health` endpoint. Expect JSON with `status: healthy` and `environment: dev`.

## 8. Local Cloud Shell exercise (alternative to GitHub Actions)
From the project directory:
```bash
bash scripts/validate.sh dev
bash scripts/deploy.sh dev   # prompts before deployment
bash scripts/deploy-app.sh dev
bash scripts/verify.sh dev
```
For automation only: `AUTO_APPROVE=true bash scripts/deploy.sh dev`.

## 9. QA and PROD
Create matching GitHub environments (`qa`, `prod`), matching Azure resource groups and Contributor assignments, and **separate** federated credentials with `environment:qa` and `environment:prod`. Configure required reviewers on `prod` GitHub environment. Run DEV first. Avoid running QA/PROD unless your lab budget permits.

## 10. Cleanup
From Cloud Shell in the project directory: `bash scripts/cleanup.sh dev`. Type the exact resource-group name to confirm deletion. Verify with `az group exists -n rg-lab11-p16-dev`. Repeat for QA/PROD only if deployed. Cleanup permanently deletes resources in that group.

## Troubleshooting
- `AADSTS700213`: federated subject/repository/environment mismatch; compare GitHub OIDC log and Entra credential.
- `Permission denied` on `validate.sh`: updated `deploy.sh` uses `bash`; workflow also sets executable bits.
- `Zip file structure invalid`: updated `deploy-app.sh` writes ZIP inside a new temporary directory and tests it with `unzip -t`.
- `AuthorizationFailed`: confirm service principal Contributor assignment on the target resource group, correct subscription and tenant.
- `No web app found`: ensure Bicep parameters set `deployApp=true` and infrastructure deployed successfully.
- `App service SKU unavailable`: verify region quota and plan availability; B1 incurs charges.
- `404/503` after deploy: check App Service logs, Node startup command and `/health` endpoint; wait for startup and retry.
