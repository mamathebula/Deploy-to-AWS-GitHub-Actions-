# Deploy to AWS (GitHub Actions)

GitHub Actions workflow that automatically deploys your CloudFormation templates to AWS every time you push changes to the `main` branch.

## How It Works

```
You edit a .yaml file in your IDE (Kiro / VS Code)
        │
        ▼
   git add . → git commit → git push
        │
        ▼
   GitHub Actions triggers automatically
        │
        ▼
   Workflow runs on a free Ubuntu server:
     1. Checks out your code
     2. Logs into AWS using your secrets
     3. Deploys each CloudFormation template
        │
        ▼
   Stacks created/updated in AWS (us-west-1)
```

## What Gets Deployed

| Stack Name | Template File | Description |
|------------|---------------|-------------|
| `aws-resource-cleanup-v2` | `aws-resource-cleanup-v2.yaml` | Automated cleanup of unused resources with cost tracking and CloudWatch dashboard |
| `ec2-instance-cleanup` | `ec2-instance-cleanup.yaml` | Terminates EC2 instances after configurable time |
| `iam-least-privilege-scanner` | `iam-least-privilege-scanner.yaml` | Weekly scan of IAM roles for overly permissive policies |

## When Does It Run?

The workflow triggers when:

- You push to the `main` branch
- AND the push includes changes to `.yaml` files

It does NOT run when you only change `.md`, `.sh`, or other non-YAML files.

The file `cost-optimization-dashboard.yaml` is excluded — changes to it will not trigger a deploy.

## Prerequisites

### 1. GitHub Repository

Your code must be in a GitHub repository. See `GIT-TO-AWS-GUIDE.md` for setup instructions.

### 2. IAM User for GitHub Actions

1. Go to AWS Console → IAM → Users → Create user
2. User name: `github-actions-deployer`
3. Attach policy: `AdministratorAccess` (or a custom policy scoped to CloudFormation + the services your templates use)
4. Create the user → Security credentials → Create access key
5. Select "Third-party service" → copy the Access Key ID and Secret Access Key

### 3. GitHub Secrets

Go to your GitHub repository → Settings → Secrets and variables → Actions → New repository secret:

| Secret Name | Value | Used For |
|-------------|-------|----------|
| `AWS_ACCESS_KEY_ID` | Your IAM user access key | Authenticating with AWS |
| `AWS_SECRET_ACCESS_KEY` | Your IAM user secret key | Authenticating with AWS |
| `NOTIFICATION_EMAIL` | Your email address | SNS notifications from all 3 stacks |

## Configuration

### Region

The workflow deploys to `us-west-1` by default. To change it, edit the `env` section in `.github/workflows/deploy-to-aws.yml`:

```yaml
env:
  AWS_REGION: eu-west-1   # change to your region
```

### Parameters

Each stack is deployed with parameter overrides. To change them, edit the `--parameter-overrides` section for each stack in the workflow file.

| Stack | Parameter | Default | Description |
|-------|-----------|---------|-------------|
| Resource Cleanup v2 | `MinRunningTimeHours` | `8` | Hours before resources are deleted |
| Resource Cleanup v2 | `ExcludeString` | `do-not-delete` | Tag value that protects resources |
| All stacks | `NotificationEmail` | From `NOTIFICATION_EMAIL` secret | Email for alerts |

### Adding a New Stack

To deploy an additional CloudFormation template, add a new step to the workflow file:

```yaml
      - name: Deploy My New Stack
        run: |
          aws cloudformation deploy \
            --template-file my-new-template.yaml \
            --stack-name my-new-stack \
            --parameter-overrides \
              NotificationEmail=${{ secrets.NOTIFICATION_EMAIL }} \
            --capabilities CAPABILITY_NAMED_IAM \
            --no-fail-on-empty-changeset
```

### Excluding a Template from Deploy

To prevent a `.yaml` file from triggering the workflow, add it to the `paths` exclusion list:

```yaml
on:
  push:
    branches:
      - main
    paths:
      - '*.yaml'
      - '!cost-optimization-dashboard.yaml'    # already excluded
      - '!my-other-template.yaml'              # add more like this
```

## How to Use

### First Time Setup

```bash
git add .github/workflows/deploy-to-aws.yml
```

```bash
git commit -m "Add GitHub Actions deploy workflow"
```

```bash
git push
```

### Day-to-Day Workflow

```bash
git pull
```

```bash
git add .
```

```bash
git commit -m "updated cleanup timer to 12 hours"
```

```bash
git push
```

The workflow runs automatically after the push. No manual steps needed.

### Watch It Run

1. Go to your GitHub repository
2. Click the Actions tab
3. Click the running workflow to see logs for each step
4. Green checkmark = deployed successfully
5. Red X = something failed — click to see the error

## Verify Deployment

After the workflow completes, verify in AWS:

### AWS Console

1. Go to CloudFormation → Stacks
2. Check that your stacks show `CREATE_COMPLETE` or `UPDATE_COMPLETE`
3. First deploy only: check your email for the SNS subscription confirmation and click the confirm link

### AWS CLI

```bash
aws cloudformation describe-stacks \
  --query "Stacks[*].[StackName,StackStatus]" \
  --output table \
  --region us-west-1
```

## What Does `--no-fail-on-empty-changeset` Mean?

If you push but the template hasn't actually changed, CloudFormation would normally throw an error saying "no changes to deploy." This flag tells it to skip and succeed instead. This prevents the workflow from failing when you push changes to one template but not the others.

## What Gets Deployed (Resources)

The workflow file itself creates nothing in AWS. It just runs `aws cloudformation deploy` for each template. The templates create the actual resources:

| Template | Resources Created |
|----------|-------------------|
| `aws-resource-cleanup-v2.yaml` | 2 Lambda functions, IAM role, SNS topic, 2 EventBridge rules, CloudWatch dashboard |
| `ec2-instance-cleanup.yaml` | 1 Lambda function, IAM role, SNS topic, EventBridge rule |
| `iam-least-privilege-scanner.yaml` | 1 Lambda function, IAM role, SNS topic, EventBridge rule |

## Cost

| Resource | Cost |
|----------|------|
| GitHub Actions | Free for public repos. Private repos get 2,000 minutes/month free |
| Workflow runs | Each run takes ~2–5 minutes. Deploying 3 stacks daily = ~150 minutes/month — well within free tier |
| AWS resources | See individual stack READMEs for cost breakdowns |

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Workflow didn't run | No `.yaml` files changed in the push | Only `.yaml` changes trigger the workflow. Check the `paths` filter |
| Workflow didn't run | Push was to a branch other than `main` | Only pushes to `main` trigger the workflow |
| `Configure AWS credentials` failed | Secrets not set or incorrect | Go to GitHub → Settings → Secrets and verify `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` |
| `No changes to deploy` in logs | Template hasn't changed since last deploy | This is normal — `--no-fail-on-empty-changeset` makes it succeed anyway |
| `ROLLBACK_COMPLETE` in AWS | Template has an error | Check CloudFormation Events tab for the specific error. Delete the failed stack and fix the template |
| `InsufficientCapabilitiesException` | Missing `--capabilities` flag | Already included in the workflow. If adding a new stack, make sure to include `--capabilities CAPABILITY_NAMED_IAM` |
| Stack stuck in `UPDATE_IN_PROGRESS` | Previous deploy still running | Wait for it to finish, or cancel the update in CloudFormation console |
| Wrong region | `AWS_REGION` doesn't match where you want to deploy | Edit the `env.AWS_REGION` value in the workflow file |
| Email notifications not arriving | SNS subscription not confirmed | Check your inbox (and spam) for the AWS confirmation email. Click the confirm link |

## Security Notes

- AWS credentials are stored as GitHub Secrets — they are never exposed in logs or code
- Consider using an IAM role with only the permissions needed instead of `AdministratorAccess`
- For production, consider using [OIDC authentication](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) instead of access keys — no long-lived credentials needed
- Rotate your access keys periodically

## Files

| File | Description |
|------|-------------|
| `.github/workflows/deploy-to-aws.yml` | The GitHub Actions workflow file |
| `GIT-TO-AWS-GUIDE.md` | Step-by-step guide for pushing code from IDE to GitHub to AWS |

## Disclaimer

This workflow is provided as-is for educational and operational purposes. Use at your own risk. Always review CloudFormation changesets before deploying to production. The author is not responsible for any unintended resource creation, modification, or costs resulting from automated deployments. Test in a non-production account first.
