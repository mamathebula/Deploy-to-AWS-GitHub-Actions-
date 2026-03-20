# Push Code from IDE to GitHub to AWS

Step-by-step guide for beginners to push code from your IDE (Kiro or VS Code) to GitHub, then automatically deploy CloudFormation templates to AWS using GitHub Actions.

## Part 1: Push Code from IDE to GitHub

### Step 1: Install Git

Check if Git is installed:

```bash
git --version
```

If not installed:
- Mac: `brew install git` or download from https://git-scm.com
- Windows: download from https://git-scm.com

### Step 2: Create a GitHub Account

Go to https://github.com and sign up if you don't have an account.

### Step 3: Create a New Repository on GitHub

1. Go to https://github.com/new
2. Enter a repository name (e.g., `aws-cost-optimization`)
3. Choose Public or Private
4. Do NOT check "Add a README file" (you already have files)
5. Click Create repository
6. Copy the repository URL (e.g., `https://github.com/yourusername/aws-cost-optimization.git`)

### Step 4: Connect Your Local Project to GitHub

Open the terminal in your IDE (Kiro: Terminal panel, VS Code: Ctrl+` or Cmd+`):

```bash
git init
```

```bash
git remote add origin https://github.com/yourusername/aws-cost-optimization.git
```

### Step 5: Add Your Files

```bash
git add .
```

Check what's being added:

```bash
git status
```

### Step 6: Make Your First Commit

```bash
git commit -m "Initial commit - AWS cost optimization tools"
```

### Step 7: Push to GitHub

```bash
git branch -M main
```

```bash
git push -u origin main
```

If prompted for credentials, enter your GitHub username and a Personal Access Token (not your password).

### How to Get a Personal Access Token

1. Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click "Generate new token (classic)"
3. Give it a name (e.g., `IDE access`)
4. Select scopes: check `repo` (full control of private repositories)
5. Click Generate token
6. Copy the token — use this as your password when Git asks

### Step 8: Future Pushes

After the first push, the workflow is:

**1. Pull — download the latest changes from GitHub**

```bash
git pull
```

This syncs your local files with what's on GitHub. If someone else made changes (or you edited from another machine), this brings those changes to your machine. Always do this first to avoid conflicts.

**2. Add — tell Git which files to include**

```bash
git add .
```

The `.` means "all changed files." Git doesn't automatically track your changes — you have to tell it what to include. Think of it like putting files in an envelope before mailing.

**3. Commit — save a snapshot of your changes**

```bash
git commit -m "your message here"
```

This saves your changes locally with a message describing what you did. The message helps you (and others) understand what changed later. Examples:
- `git commit -m "updated cleanup timer to 4 hours"`
- `git commit -m "added IAM scanner template"`
- `git commit -m "fixed gp2 migration script error handling"`

**4. Push — upload your commit to GitHub**

```bash
git push
```

This sends your committed changes to GitHub. Once pushed, GitHub Actions automatically triggers and deploys your CloudFormation templates to AWS.

**The full cycle visualized:**

```
git pull       →  Get latest from GitHub to your machine
     ↓
(edit files)   →  Make your changes in the IDE
     ↓
git add .      →  Stage the changed files
     ↓
git commit     →  Save snapshot locally with a message
     ↓
git push       →  Upload to GitHub → triggers deploy to AWS
```

### Using the IDE GUI (No Terminal)

#### Kiro / VS Code

1. Click the Source Control icon (branch icon) in the left sidebar
2. You'll see all changed files listed
3. Click the `+` next to each file to stage it (or `+` at the top to stage all)
4. Type a commit message in the text box
5. Click the checkmark (✓) to commit
6. Click the `...` menu → Push (or click the sync icon in the bottom status bar)

---

## Part 2: Auto-Deploy to AWS with GitHub Actions

When you push code to GitHub, GitHub Actions can automatically deploy your CloudFormation templates to AWS.

### Step 1: Create an IAM User for GitHub Actions

1. Go to AWS Console → IAM → Users → Create user
2. User name: `github-actions-deployer`
3. Click Next
4. Attach policies directly → search and select `AdministratorAccess` (or create a custom policy with only CloudFormation + the services your templates use)
5. Click Create user
6. Click on the user → Security credentials → Create access key
7. Select "Third-party service" → Next
8. Copy the Access Key ID and Secret Access Key

### Step 2: Add Secrets to GitHub

1. Go to your GitHub repository
2. Click Settings → Secrets and variables → Actions
3. Click "New repository secret" and add these three secrets:

| Secret Name | Value |
|-------------|-------|
| `AWS_ACCESS_KEY_ID` | Your IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | Your IAM user secret key |
| `NOTIFICATION_EMAIL` | Your email for SNS notifications |

### Step 3: Create the Workflow File

Create this file in your project: `.github/workflows/deploy-to-aws.yml`

```yaml
name: Deploy CloudFormation to AWS

# WHEN: triggers when you push to the main branch and a .yaml file changed
on:
  push:
    branches:
      - main        # only runs on pushes to main (not other branches)
    paths:
      - '*.yaml'    # only runs if a .yaml file was changed

# VARIABLES: shared across all jobs
env:
  AWS_REGION: us-west-1   # change this to your AWS region

jobs:
  deploy:
    runs-on: ubuntu-latest   # runs on a fresh Ubuntu server provided by GitHub (free)
    steps:

      # Step 1: Download your code from GitHub onto the Ubuntu server
      - name: Checkout code
        uses: actions/checkout@v4

      # Step 2: Set up AWS credentials so the server can talk to your AWS account
      # Uses the secrets you added in GitHub Settings → Secrets
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      # Step 3: Deploy the Resource Cleanup v2 stack
      # "deploy" creates the stack if it doesn't exist, or updates it if it does
      # --no-fail-on-empty-changeset = don't fail if nothing changed
      - name: Deploy Resource Cleanup v2
        run: |
          aws cloudformation deploy \
            --template-file aws-resource-cleanup-v2.yaml \
            --stack-name aws-resource-cleanup-v2 \
            --parameter-overrides \
              NotificationEmail=${{ secrets.NOTIFICATION_EMAIL }} \
              MinRunningTimeHours=8 \
              ExcludeString=do-not-delete \
            --capabilities CAPABILITY_NAMED_IAM \
            --no-fail-on-empty-changeset

      # Step 4: Deploy the EC2 Instance Cleanup stack
      - name: Deploy EC2 Instance Cleanup
        run: |
          aws cloudformation deploy \
            --template-file ec2-instance-cleanup.yaml \
            --stack-name ec2-instance-cleanup \
            --parameter-overrides \
              NotificationEmail=${{ secrets.NOTIFICATION_EMAIL }} \
            --capabilities CAPABILITY_NAMED_IAM \
            --no-fail-on-empty-changeset

      # Step 5: Deploy the IAM Least Privilege Scanner stack
      - name: Deploy IAM Least Privilege Scanner
        run: |
          aws cloudformation deploy \
            --template-file iam-least-privilege-scanner.yaml \
            --stack-name iam-least-privilege-scanner \
            --parameter-overrides \
              NotificationEmail=${{ secrets.NOTIFICATION_EMAIL }} \
            --capabilities CAPABILITY_NAMED_IAM \
            --no-fail-on-empty-changeset
```

### What Each Part Does

| Part | Purpose |
|------|---------|
| `name:` | Name of the workflow — shows up in the GitHub Actions tab |
| `on: push:` | Trigger — runs when code is pushed |
| `branches: main` | Only runs on the main branch, not feature branches |
| `paths: '*.yaml'` | Only runs if a `.yaml` file changed (skips README-only changes) |
| `env: AWS_REGION` | Sets the AWS region for all steps |
| `runs-on: ubuntu-latest` | GitHub provides a free Ubuntu server to run your workflow |
| `actions/checkout@v4` | Downloads your repo code onto the server |
| `configure-aws-credentials@v4` | Logs into AWS using your secrets |
| `${{ secrets.XXX }}` | Pulls values from GitHub Secrets (never exposed in logs) |
| `aws cloudformation deploy` | Creates or updates a CloudFormation stack |
| `--template-file` | Which `.yaml` file to deploy |
| `--stack-name` | Name of the stack in AWS |
| `--parameter-overrides` | Values for the template parameters |
| `--capabilities CAPABILITY_NAMED_IAM` | Required when the template creates IAM roles |
| `--no-fail-on-empty-changeset` | Don't fail if nothing changed since last deploy |

### Step 4: Push the Workflow

```bash
git add .github/workflows/deploy-to-aws.yml
```

```bash
git commit -m "Add GitHub Actions deploy workflow"
```

```bash
git push
```

### Step 5: Watch It Run

1. Go to your GitHub repository
2. Click the Actions tab
3. You'll see the workflow running
4. Click on it to see the logs for each step
5. Green checkmark = deployed successfully

### How It Works

```
You edit code in IDE (Kiro / VS Code)
        │
        ▼
   git push to GitHub (main branch)
        │
        ▼
   GitHub Actions triggers automatically
        │
        ▼
   Workflow runs on Ubuntu server:
     1. Checks out your code
     2. Configures AWS credentials (from secrets)
     3. Deploys each CloudFormation template
        │
        ▼
   Stacks created/updated in AWS
```

### When Does It Run?

The workflow triggers when:
- You push to the `main` branch
- AND the push includes changes to `.yaml` files

It does NOT run when you only change `.md`, `.sh`, or other non-YAML files.

### What Does `--no-fail-on-empty-changeset` Mean?

If you push but the template hasn't actually changed, CloudFormation would normally throw an error saying "no changes to deploy." This flag tells it to just skip and succeed instead.

---

## Part 3: Verify Deployment

After the workflow runs, verify in AWS:

1. Go to AWS Console → CloudFormation
2. Check that your stacks show `CREATE_COMPLETE` or `UPDATE_COMPLETE`
3. Check your email for the SNS subscription confirmation (first deploy only)

### Check from CLI

```bash
aws cloudformation describe-stacks \
  --query "Stacks[*].[StackName,StackStatus]" \
  --output table
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Stage all files | `git add .` |
| Stage one file | `git add filename` |
| Commit | `git commit -m "message"` |
| Push | `git push` |
| Check status | `git status` |
| See commit history | `git log --oneline` |
| Pull latest changes | `git pull` |
| See what changed | `git diff` |

## Common Issues

| Problem | Fix |
|---------|-----|
| `fatal: not a git repository` | Run `git init` first |
| `remote origin already exists` | Run `git remote remove origin` then add it again |
| `Authentication failed` | Use a Personal Access Token, not your GitHub password |
| GitHub Actions failed | Click the failed job in the Actions tab to see the error log |
| `No changes to deploy` | This is normal — the template hasn't changed since last deploy |
| Wrong region | Change `AWS_REGION` in the workflow file to your region |
