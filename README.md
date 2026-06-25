# RHPDS Reusable GitHub Workflows

This repository contains reusable GitHub Actions workflows for the RHPDS organization.

## Available Workflows

### 🤖 Jira AI Analysis

Automatically analyzes Git commits using Claude AI and posts summaries to linked Jira tickets.

**Features:**
- Reads Jira ticket from `.jira` file in your repo
- Analyzes commit diffs with Claude AI
- Posts AI-generated summaries to Jira tickets
- Runs on every push

---

## Usage

### 1. Add `.jira` file to your repository

Create a `.jira` file in your repository root containing the Jira ticket number:

```
RHCLOUD-1234
```

### 2. Create workflow file in your repository

Create `.github/workflows/jira-update.yml`:

```yaml
name: Update Jira with AI Analysis

on:
  push:
    branches:
      - main
      - master
      - develop
      # Add any other branches you want to track

jobs:
  jira-analysis:
    uses: rhpds/rhpds-workflows/.github/workflows/jira-ai-update.yml@main
    secrets:
      MAAS_API_KEY: ${{ secrets.PH_MAAS_API_TOKEN }}
      JIRA_API_TOKEN: ${{ secrets.PH_JIRA_API_TOKEN }}
      JIRA_EMAIL: ${{ secrets.PH_JIRA_API_USER }}
```

### 3. Commit and push

That's it! On your next commit, the workflow will:
1. Check for `.jira` file
2. Get the commit diff
3. Analyze it with Claude AI
4. Post a summary to your Jira ticket

---

## Organization Secrets

The following secrets are configured at the organization level and available to all repos:

- `PH_MAAS_API_TOKEN` - API key for MAAS Claude endpoint
- `PH_JIRA_API_TOKEN` - Jira API token
- `PH_JIRA_API_USER` - Jira user email

---

## Example Jira Comment

When a commit is pushed, Jira receives a comment like:

> 🤖 **AI Analysis of commit `abc123`**
> 
> **Repository:** rhpds/my-project  
> **Commit:** https://github.com/rhpds/my-project/commit/abc123
> 
> ---
> 
> **Summary:** Added input validation to login form to prevent empty username/password submissions.
> 
> **Impact:** Improves security by rejecting invalid login attempts before they reach the backend.
> 
> **Testing:** Verify empty fields show validation errors, test with various invalid inputs.

---

## Skipping Jira Integration

If a repository doesn't have a `.jira` file, the workflow automatically skips Jira integration (no errors).

---

## Updating the Jira Ticket

When you switch to working on a different Jira ticket, simply update the `.jira` file:

```bash
echo "RHCLOUD-5678" > .jira
git add .jira
git commit -m "Switch to new Jira ticket"
git push
```

All future commits will post to the new ticket.

---

## Troubleshooting

**Workflow not running?**
- Check that `.github/workflows/jira-update.yml` exists in your repo
- Verify the workflow references `@main` branch of this repo

**Jira not updating?**
- Verify `.jira` file contains a valid ticket number
- Check the Actions tab in your repo for error logs
- Ensure organization secrets are configured

**AI analysis failing?**
- Check that MAAS endpoint is accessible from GitHub runners
- Verify `PH_MAAS_API_TOKEN` secret is valid

---

## Contributing

To modify the reusable workflow:
1. Make changes to `.github/workflows/jira-ai-update.yml`
2. Test with a demo repository
3. Push to `main` branch - all repos using it will automatically get the update
