# LAB 11 — Project 16: Azure Application Deployment Pipeline (Student Edition)

Deploy a Node.js health-check application to Azure App Service with modular Bicep, Application Insights, and GitHub Actions OIDC. Region defaults to `centralindia`; standard Azure subscription required.

**Start here:** [STUDENT-SETUP.md](STUDENT-SETUP.md). Instructor's original labs, test scenarios, operations, diagrams, and application source remain included.

## Fixes incorporated
1. Bicep modules reformatted into valid multiline syntax (BCP236 fix).
2. `deploy.sh` invokes validation via Bash and honors `AUTO_APPROVE=true`.
3. `deploy-app.sh` creates a new ZIP in a temporary directory and verifies it with `unzip -t` before Azure deployment.
4. `verify.sh` checks actual deployed App Service hostname and `/health`.
5. Reusable workflow template in `github-workflow/azure-deploy.yml` for copying to **repository root** `.github/workflows/azure-deploy.yml`.
6. Each student must configure their own GitHub fork, OIDC credential, app registration, RBAC, and three Azure IDs.

## Local commands
```bash
az account set --subscription '<YOUR_SUBSCRIPTION_ID>'
bash scripts/validate.sh dev
bash scripts/deploy.sh dev
bash scripts/deploy-app.sh dev
bash scripts/verify.sh dev
bash scripts/cleanup.sh dev
```

**Cost warning:** B1 Linux App Service and monitoring are billable. Do not leave lab resources running unnecessarily. Production deployments should require approval.
