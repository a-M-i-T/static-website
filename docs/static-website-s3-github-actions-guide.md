# Deploying a Static Website to Amazon S3 with GitHub Actions

## Overview

I set up this deployment so that I can update my static website by pushing changes to GitHub. GitHub Actions runs the deployment workflow and syncs the website files from the repository root to my Amazon S3 bucket.

**Deployment flow**

`Local changes → GitHub (main branch) → GitHub Actions → Amazon S3`

This project uses AWS Academy temporary credentials. The workflow worked with the credentials available during setup, but those credentials expire and may need to be refreshed.

## 1. What I needed

- A GitHub repository containing the static website files.
- An Amazon S3 bucket for the website files.
- AWS CLI installed locally (useful for checking access and testing S3).
- Permission to use the AWS Academy credentials in GitHub Actions.
- A GitHub Actions workflow stored in `.github/workflows/deploy.yml`.

The website files for this setup are in the **root of the repository**. The workflow therefore syncs `./` rather than a `dist` or `build` directory.

## 2. Check AWS access

I used the AWS Academy Learner Lab credentials from the AWS Details section. These are temporary credentials, so all three values are needed:

- Access key ID
- Secret access key
- Session token

I configured the AWS CLI locally and checked that it could identify the active AWS session:

```bash
aws sts get-caller-identity
```

I also checked the S3 bucket. An empty result from `aws s3 ls` can simply mean that the bucket has no objects; it does not by itself prove that uploads are allowed. A small test upload with `aws s3 cp` is a better way to check write access.

> Do not put AWS credentials in the repository, workflow YAML, screenshots, or commit history. If credentials are accidentally exposed, stop using them and follow the AWS Academy/instructor process for replacing them.

## 3. Add GitHub Actions secrets

In the GitHub repository, I opened:

**Settings → Secrets and variables → Actions → New repository secret**

I added these repository secrets:

| Secret name | What it contains |
|---|---|
| `AWS_ACCESS_KEY_ID` | AWS Academy access key ID |
| `AWS_SECRET_ACCESS_KEY` | AWS Academy secret access key |
| `AWS_SESSION_TOKEN` | AWS Academy session token |
| `AWS_REGION` | Region where the S3 bucket is located, for example `ap-south-1` |
| `S3_BUCKET` | Exact name of the destination S3 bucket |

The secret values are supplied to the workflow at runtime. They are not written directly into the YAML file.

## 4. Create the deployment workflow

I created this file in the repository:

```text
.github/
└── workflows/
    └── deploy.yml
```

Workflow content:

```yaml
name: Deploy Website to Amazon S3

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v6.3.0
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Verify AWS identity
        run: aws sts get-caller-identity

      - name: Deploy website to S3
        run: |
          aws s3 sync ./ s3://${{ secrets.S3_BUCKET }}/ \
            --exclude ".git/*" \
            --exclude ".github/*" \
            --exclude "node_modules/*" \
            --exclude ".env*"
```

### What the workflow does

1. **Checkout repository:** downloads the latest code from the repository.
2. **Configure AWS credentials:** makes the GitHub secrets available to the AWS credentials action.
3. **Verify AWS identity:** checks which AWS identity the runner is using. It does not print the secret access key or session token.
4. **Sync files to S3:** uploads new and changed files from the repository root to the bucket.

The workflow runs on pushes to the `main` branch. It deliberately does not use `--delete`, so files already in the bucket are not removed just because they are absent from the repository. Review the bucket contents before changing that behaviour.

## 5. Run and verify a deployment

After committing and pushing the workflow file to `main`:

1. Open the repository on GitHub.
2. Select the **Actions** tab.
3. Open the **Deploy Website to Amazon S3** workflow run.
4. Check each step. A successful run should show green check marks.
5. Open the S3 bucket in the AWS Console and confirm that the website files, including `index.html`, are present.
6. If static website hosting is enabled, open the bucket website endpoint and check that the site loads.

A successful workflow run confirms that the workflow completed. Checking the S3 objects and opening the website endpoint provides an additional end-to-end check.

## 6. Updating the website

For a normal update:

1. Edit the website files locally.
2. Test the changes.
3. Commit and push them to the `main` branch.
4. Open **Actions** and check the new deployment run.
5. Refresh the website endpoint to confirm the update.

GitHub Actions handles the S3 sync after the push; there is no need to run the upload command manually for each update.

## 7. AWS Academy credential limitation

The deployment uses AWS Academy temporary credentials because the Learner Lab role did not allow creation of the GitHub Actions OIDC identity provider. The credentials can expire when the lab session ends or when AWS Academy refreshes them.

If a later deployment fails with an authentication or expired-token error:

1. Start or resume the Learner Lab.
2. Copy the current access key, secret access key, and session token from AWS Details.
3. Update the matching GitHub repository secrets.
4. Run the workflow again.

Do not assume that old credentials will continue working. Do not commit credentials to GitHub or place them in the workflow file.

For a longer-term deployment, an instructor/admin-approved OIDC setup with a narrowly scoped IAM role is preferable. The current Learner Lab role may not have permission to create the identity provider or role, so this needs to be arranged through the course administrator rather than worked around.

## 8. Troubleshooting notes

| Symptom | What to check |
|---|---|
| `InvalidClientTokenId` or expired-token message | Refresh all three AWS credential secrets, including the session token. |
| `AccessDenied` during sync | Check that the lab role is allowed to upload objects to this bucket. Do not try to bypass the lab restrictions. |
| Workflow cannot find the bucket | Check `S3_BUCKET` spelling and `AWS_REGION`. |
| Workflow succeeds but site is unchanged | Confirm the latest commit is on `main`, check the sync step's log, and confirm the bucket/endpoint being opened is the intended one. |
| `index.html` is missing | Confirm it is in the repository root. If the site is built into another directory, change the sync source to that directory. |

## 9. Final setup summary

- **Source:** GitHub repository, root directory
- **Trigger:** push to `main`
- **Automation:** GitHub Actions
- **Destination:** Amazon S3 bucket
- **Authentication:** AWS Academy temporary credentials stored as GitHub repository secrets
- **Sync command:** `aws s3 sync ./ s3://<bucket-name>/`
- **Important limitation:** credentials expire and may need to be updated

This setup gives me a simple way to publish static website changes by pushing them to GitHub. It is suitable for the current AWS Academy environment, with the temporary-credential limitation kept in mind.
