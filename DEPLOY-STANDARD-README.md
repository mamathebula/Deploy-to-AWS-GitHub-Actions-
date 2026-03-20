# Deploy to AWS Standard (GitHub Actions Template)

Reusable GitHub Actions workflow template for deploying CloudFormation templates to AWS. Copy it, change the region and stack names, and push.

## File

`deploy-to-aws-standard.yml` — copy this to `.github/workflows/deploy-to-aws.yml` in any project.

## How to Use

### Step 1: Copy the File

```bash
mkdir -p .github/workflows
```

```bash
cp deploy-to-aws-standard.yml .github/workflows/deploy-to-aws.yml
```

### Step 2: Change the Region

Open `.github/workflows/deploy-to-aws.yml` and change:

```yaml
env:
  AWS_REGION: us-west-1   # ← change to your region
```

### Step 3: Add Your Stacks

For each CloudFormation template you want to deploy, add a step:

```yaml
      - name: Deploy My Stack
        run: |
          aws cloudformation deploy \
            --template-file my-stack.yaml \
            --stack-name my-stack \
            --parameter-overrides \
              NotificationEmail=${{ secrets.NOTIFICATION_EMAIL }} \
            --capabilities CAPABILITY_NAMED_IAM \
            --no-fail-on-empty-changeset
```

Change these values:

| Value | What to Change |
|-------|---------------|
| `name:` | Description of the stack (shows in GitHub Actions logs) |
| `--template-file` | Your `.yaml` file name |
| `--stack-name` | Name for the stack in AWS (lowercase-with-dashes) |
| `--parameter-overrides` | Values for your template's Parameters section |

### Step 4: Add GitHub Secrets

Go to your GitHub repository → Settings → Secrets and variables → Actions → New repository secret:

| Secret Name | Value |
|-------------|-------|
| `AWS_ACCESS_KEY_ID` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key |
| `NOTIFICATION_EMAIL` | Your email for SNS notifications |

### Step 5: Push

```bash
git add .github/workflows/deploy-to-aws.yml
```

```bash
git commit -m "Add deploy workflow"
```

```bash
git push
```

The workflow runs automatically when you push `.yaml` changes to the `main` branch.

## Exclude Templates from Auto-Deploy

If you have `.yaml` files that should NOT trigger a deploy (e.g., templates you deploy manually), add them to the exclusion list:

```yaml
on:
  push:
    branches:
      - main
    paths:
      - '*.yaml'
      - '!my-manual-template.yaml'     # ← won't trigger deploy
      - '!another-template.yaml'        # ← add more like this
```

## Add Parameters

If your template has extra parameters beyond `NotificationEmail`, add them to `--parameter-overrides`:

```yaml
      - name: Deploy My Stack
        run: |
          aws cloudformation deploy \
            --template-file my-stack.yaml \
            --stack-name my-stack \
            --parameter-overrides \
              NotificationEmail=${{ secrets.NOTIFICATION_EMAIL }} \
              MinRunningTimeHours=8 \
              ExcludeString=do-not-delete \
              MyCustomParam=my-value \
            --capabilities CAPABILITY_NAMED_IAM \
            --no-fail-on-empty-changeset
```

For sensitive parameters, store them as GitHub Secrets and reference with `${{ secrets.SECRET_NAME }}`.

## Capabilities Reference

| Capability | When to Use |
|------------|-------------|
| `CAPABILITY_NAMED_IAM` | Template creates IAM roles with custom names (most common) |
| `CAPABILITY_IAM` | Template creates IAM roles without custom names |
| `CAPABILITY_AUTO_EXPAND` | Template uses macros or transforms (e.g., SAM) |

If unsure, use `CAPABILITY_NAMED_IAM` — it covers both named and unnamed IAM resources.

## Common Regions

| Region | Location |
|--------|----------|
| `us-east-1` | N. Virginia |
| `us-west-1` | N. California |
| `us-west-2` | Oregon |
| `eu-west-1` | Ireland |
| `eu-central-1` | Frankfurt |
| `ap-southeast-1` | Singapore |
| `ap-southeast-2` | Sydney |
| `ap-northeast-1` | Tokyo |

## Example: Current Project

The current project's workflow (`.github/workflows/deploy-to-aws.yml`) deploys 3 stacks:

| Stack | Template | Parameters |
|-------|----------|------------|
| `aws-resource-cleanup-v2` | `aws-resource-cleanup-v2.yaml` | `NotificationEmail`, `MinRunningTimeHours=8`, `ExcludeString=do-not-delete` |
| `ec2-instance-cleanup` | `ec2-instance-cleanup.yaml` | `NotificationEmail` |
| `iam-least-privilege-scanner` | `iam-least-privilege-scanner.yaml` | `NotificationEmail` |

## Cost

| Resource | Cost |
|----------|------|
| GitHub Actions | Free for public repos. Private repos: 2,000 minutes/month free |
| Each workflow run | ~2–5 minutes |
| AWS resources | Depends on what your templates create (see individual stack READMEs) |

## Related Files

| File | Description |
|------|-------------|
| `deploy-to-aws-standard.yml` | The reusable workflow template (copy this) |
| `.github/workflows/deploy-to-aws.yml` | The active workflow for this project |
| `CLOUDFORMATION-STANDARD.md` | Standard template for creating new CloudFormation `.yaml` files |
| `GIT-TO-AWS-GUIDE.md` | Full guide for pushing code from IDE to GitHub to AWS |
| `DEPLOY-TO-AWS-README.md` | README for the current project's deploy workflow |
