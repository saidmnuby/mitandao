# Admin Deployment Workflow Guide

**Version:** 1.0  
**Updated:** 21 November 2026

## 1. What the deployment system does

The admin controls the deployment process from the GitHub repository `saidmnuby/mitandao`.

When a developer pushes to `main` in their assigned repository:

1. The developer repository sends a deployment request to the admin repository.
2. The admin workflow checks the project ID in `deploy-projects.json`.
3. It downloads the approved repository commit using a read-only source token.
4. It uploads eligible files to the assigned cPanel folder for that subdomain.
5. The admin checks the deployment result in GitHub Actions.

The developer does not receive cPanel access or the cPanel API token.

## 2. One-time admin setup

In `saidmnuby/mitandao`, keep these files:

- `.github/workflows/deploy.yml` — the central deployment workflow.
- `deploy-projects.json` — the approved projects and their settings.

In **Settings → Secrets and variables → Actions**, store:

- `CPANEL_HOST` — the cPanel server host name.
- `CPANEL_USER` — the cPanel account username.
- `CPANEL_API_TOKEN` — the cPanel API token.
- One read-only GitHub token for each private developer repository. The matching secret name is listed in `deploy-projects.json`, for example `SOURCE_READ_TOKEN`.

A source token needs read access to the developer repository's contents only. Do not put cPanel credentials in a developer repository.

## 3. Add a developer project

For each new developer:

1. Create or confirm the developer's private GitHub repository.
2. Create the subdomain and confirm its document-root folder in cPanel.
3. Create a read-only token that can read that repository's contents. Save it as an Actions secret in `saidmnuby/mitandao`.
4. Add one project entry to `deploy-projects.json`:

```json
{
  "project_id": {
    "repository": "developer-account/repository-name",
    "branch": "main",
    "target": "/home/elimutai/subdomain.example.com",
    "source_token": "SOURCE_READ_TOKEN_PROJECT_ID"
  }
}
```

Use a unique project ID and secret name for each project. The `target` must be the approved document-root folder. Do not accept a target path supplied by the developer workflow.

5. Give the developer the standard trigger workflow. They change only the project ID and add the `DEPLOY_TRIGGER_TOKEN` secret to their repository.
6. Ask the developer to push a small approved change to `main` and monitor both workflow runs.

## 4. Review and monitor a deployment

There are two workflow runs:

- The developer repository's `Request hosting deployment` run only sends the request.
- The admin repository's `Deploy approved developer project` run performs the upload.

A successful request run does not prove that files reached cPanel. Confirm that the admin deployment run is green and that its log lists the files uploaded or updated. Then open the assigned subdomain and check the change.

If the admin run fails, read the error in the upload step. Do not ask the developer to send credentials or tokens. Share only the error text or run link needed for diagnosis.

## 5. Deployment file rules

The workflow uploads eligible files from the repository root (`./`). It skips common development or private files, including `.git`, `.github`, `node_modules`, `.env*`, `.cpanel.yml`, README and common documentation files, `.gitignore`, `.gitattributes`, `.editorconfig`, `.DS_Store`, and `Thumbs.db`.

The deployment is intended to overwrite files with matching names and create needed folders. The current workflow does not automatically delete files that were removed or renamed in GitHub. Review the destination before removing old files manually, especially if the folder may contain files from another project.

This workflow uploads repository files. It does not install project dependencies or run a build command. If a project needs a build step, confirm and add that step before onboarding it.

## 6. Security rules for the admin

- Keep `CPANEL_API_TOKEN` and source read tokens only in admin repository secrets.
- Give each source token access to only its own developer repository, with contents read permission.
- Keep destination paths fixed in `deploy-projects.json` and validate them in the central workflow.
- Protect the admin repository's `main` branch. Review changes to the workflow and project list before merging.
- Do not deploy a project until its owner, repository, subdomain, and document root are confirmed.
- Keep a record of each project's repository, project ID, subdomain, target folder, and token secret name. Never record token values.

A developer's `DEPLOY_TRIGGER_TOKEN` can request that the admin workflow run. The approved project map controls which repository and destination the admin workflow uses. Keep the admin workflow fixed to those approved mappings.

## 7. Pause or remove a project

To pause a project, remove or disable its developer trigger and remove its project ID from `deploy-projects.json`. Revoke its source read token and its trigger token when access is no longer needed. Check the assigned hosting folder before removing files; the workflow does not clean up old files automatically.

Pause deployment access immediately if a token is exposed, a workflow is changed to bypass the approved project mapping, a developer targets another project, or deployed content creates a security or service risk. Investigate the Actions logs, rotate affected tokens, and restore access only after the issue is resolved.
## 8. Where credentials come from and where they go

There are three separate credentials. They have different jobs and must not be mixed.

| Secret | Created in | Stored in | Used for |
|---|---|---|---|
| `CPANEL_API_TOKEN` | cPanel account `elimutai` | Admin repository `saidmnuby/mitandao` | Allows the admin workflow to upload files and create folders in cPanel. |
| `SOURCE_READ_TOKEN` (use a separate secret name for each private source repository) | GitHub account `saidmnuby` | Admin repository `saidmnuby/mitandao` | Allows the admin workflow to read the assigned developer repository during checkout. |
| `DEPLOY_TRIGGER_TOKEN` | GitHub account `saidmnuby` | Each developer repository, as an Actions secret | Allows the developer workflow to request a run of the admin workflow. |

### Create or find each value

**`CPANEL_HOST`**  
Use the cPanel server host name given by the hosting provider. For this setup it is `hm105.radns.net`. The admin workflow connects to `https://hm105.radns.net:2083`.

**`CPANEL_USER`**  
Use the cPanel account username given by the hosting provider. For this setup it is `elimutai`.

**`CPANEL_API_TOKEN`**  
Sign in to cPanel as `elimutai`. Open **Security → Manage API Tokens**, create a token, and copy it once. Save it in `saidmnuby/mitandao` under **Settings → Secrets and variables → Actions** as `CPANEL_API_TOKEN`. The workflow sends it to the cPanel server over HTTPS in the cPanel authorization header. Never put this token in a developer repository.

**Source read token**  
While signed in to the admin GitHub account `saidmnuby`, create a fine-grained personal access token. Select only the developer repository it must read and grant **Contents: Read-only**. Save it as an Actions secret in `saidmnuby/mitandao`; use the exact secret name written for that project in `deploy-projects.json`. The admin workflow uses it only to check out that repository's selected commit.

**Trigger token**  
While signed in to `saidmnuby`, create a separate fine-grained personal access token. Select only the admin repository `saidmnuby/mitandao` and grant **Actions: Read and write**. Save it in the developer repository as an Actions secret named `DEPLOY_TRIGGER_TOKEN`. The developer workflow sends it to GitHub's workflow-dispatch API. It does not give the developer the cPanel token.

### How a deployment request travels

```text
Developer pushes to main
        ↓
Developer Actions uses DEPLOY_TRIGGER_TOKEN
        ↓
GitHub receives a request to start the admin workflow
        ↓
Admin workflow checks deploy-projects.json
        ↓
Admin workflow uses that project's SOURCE_READ_TOKEN to download its commit
        ↓
Admin workflow uses CPANEL_HOST, CPANEL_USER, and CPANEL_API_TOKEN
        ↓
cPanel receives the files at the approved target folder
```

The admin workflow calls cPanel over HTTPS on port `2083`. It uses cPanel's `Fileman::upload_files` endpoint to upload files and its directory-creation endpoint when needed. The upload request includes the target folder and file data; the cPanel API token stays in the admin workflow's environment and is not sent to the developer repository.

**Token scope note:** A `DEPLOY_TRIGGER_TOKEN` with Actions write access to the admin repository can request the admin workflow to run. The admin workflow must validate the project ID against `deploy-projects.json` and choose the source repository and destination from that file. Never accept a destination path from the developer workflow.