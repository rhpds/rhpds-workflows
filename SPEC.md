# GitHub to Jira AI Integration Specification

## Overview

Automated workflow system that validates Jira ticket references in pull requests and posts AI-generated code analysis to Jira tickets when PRs are merged.

**Version:** 1.0  
**Last Updated:** 2026-06-25  
**Organization:** RHPDS (Red Hat Demo Platform)

---

## Architecture

### Components

1. **Validation Workflow** (`validate-jira-ticket.yml`)
   - Runs on PR creation and updates
   - Validates presence of Jira ticket in PR description
   - Fails if no valid ticket found
   - Can be set as required check to block merges

2. **AI Analysis Workflow** (`jira-ai-update.yml`)
   - Runs ONLY when PR is merged
   - Extracts full PR diff using commit SHAs
   - Sends diff to Claude AI via MAAS API for analysis
   - Posts structured analysis to linked Jira ticket

3. **PR Template** (`.github/pull_request_template.md`)
   - Auto-fills PR description
   - Includes required Jira ticket field
   - Provides checklist and structure

### Data Flow

```
PR Created → Validation Workflow
             ├─ Extract Jira ticket from description
             ├─ Validate format (e.g., GPTEINFRA-1234)
             └─ ✅ Pass / ❌ Fail

PR Merged → AI Analysis Workflow
            ├─ Get PR diff (base SHA → head SHA)
            ├─ Send to Claude AI (MAAS endpoint)
            ├─ Receive analysis (summary, risks, testing)
            └─ Post to Jira ticket via API
```

---

## Setup Instructions

### Prerequisites

- GitHub Enterprise organization
- Red Hat Jira access (redhat.atlassian.net)
- MAAS API access (maas-rhdp.apps.maas.redhatworkshops.io)
- Jira bot account with comment permissions

### 1. Organization-Level Secrets

Configure in GitHub Organization Settings → Secrets:

| Secret Name | Value | Description |
|-------------|-------|-------------|
| `PH_MAAS_API_TOKEN` | `ATATT3xFf...` | MAAS API authentication token |
| `PH_JIRA_API_TOKEN` | `ATATT3xFf...` | Jira bot account API token |
| `PH_JIRA_API_USER` | `rhdp-jira-bot@redhat.com` | Jira bot email address |

### 2. Per-Repository Setup

#### Add Workflow Files

Create `.github/workflows/jira-update.yml`:

```yaml
name: Jira Integration

on:
  pull_request:
    types: [opened, edited, synchronize, reopened, closed]

jobs:
  # Validate Jira ticket when PR is opened or updated
  validate-ticket:
    if: github.event.action != 'closed'
    uses: rhpds/rhpds-workflows/.github/workflows/validate-jira-ticket.yml@main

  # Run AI analysis and update Jira when PR is MERGED
  jira-analysis:
    if: github.event.pull_request.merged == true
    uses: rhpds/rhpds-workflows/.github/workflows/jira-ai-update.yml@main
    secrets:
      MAAS_API_KEY: ${{ secrets.PH_MAAS_API_TOKEN }}
      JIRA_API_TOKEN: ${{ secrets.PH_JIRA_API_TOKEN }}
      JIRA_EMAIL: ${{ secrets.PH_JIRA_API_USER }}
```

#### Add PR Template

Create `.github/pull_request_template.md`:

```markdown
## Description
<!-- Provide a brief description of the changes in this PR -->


## Jira Ticket
**Jira:** REPLACE_WITH_TICKET_NUMBER


## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
<!-- Describe how you tested these changes -->


## Checklist
- [ ] My code follows the project's style guidelines
- [ ] I have performed a self-review of my code
- [ ] I have commented my code where necessary
- [ ] I have updated the documentation accordingly
- [ ] My changes generate no new warnings
- [ ] I have added tests that prove my fix is effective or that my feature works
```

#### Optional: Make Validation Required

To enforce Jira tickets before merge:

1. Go to repository Settings → Branches → Branch protection rules
2. Edit rule for `main` branch
3. Enable "Require status checks to pass before merging"
4. Search for and select: `validate-ticket / validate`
5. Save

---

## Workflow Behavior

### Validation Workflow

**Triggers:**
- PR opened
- PR edited (description changed)
- PR synchronized (new commits pushed)
- PR reopened

**Does NOT trigger:**
- PR closed/merged

**Logic:**
1. Extracts PR description body
2. Searches for pattern: `**Jira:** TICKET-NUMBER`
3. Validates format: `[A-Z]+-[0-9]+`
4. Passes if ticket found, fails otherwise

**Output (Success):**
```
════════════════════════════════════════════════
✅ VALIDATION PASSED
════════════════════════════════════════════════

🎫 Found Jira ticket: GPTEINFRA-1234

════════════════════════════════════════════════
```

**Output (Failure):**
```
════════════════════════════════════════════════
❌ VALIDATION FAILED
════════════════════════════════════════════════

⚠️  No Jira ticket found in PR description

📝 Please update your PR description to include:
   **Jira:** GPTEINFRA-1234

🔗 Edit your PR here:
   https://github.com/rhpds/repo/pull/123

════════════════════════════════════════════════
```

### AI Analysis Workflow

**Triggers:**
- PR merged to base branch

**Does NOT trigger:**
- PR opened/updated (before merge)
- PR closed without merge

**Logic:**

1. **Extract Jira Ticket**
   - Reads from PR description
   - Fallback: checks for `.jira` file (legacy)
   - Skips if no ticket found

2. **Get PR Diff**
   - Uses `base.sha` and `head.sha` from PR event
   - Captures full PR changes (before merge state)
   - Format: `git diff base_sha...head_sha`

3. **Analyze with AI**
   - Sends diff to MAAS Claude API
   - Model: `claude-opus-4-6`
   - Max tokens: 1000
   - Prompt requests: summary, impact/risks, testing recommendations

4. **Post to Jira**
   - Uses Jira REST API v3
   - Format: ADF (Atlassian Document Format)
   - Bot account credentials
   - Includes: PR number, title, author, repository, AI analysis

**Example Jira Comment:**

```
🤖 AI Analysis of PR #42

Repository: rhpds/jira-ai-demo
Author: @treddy08
PR: Add error handling to login validation

────────────────────────────────────

Summary: This PR adds comprehensive error handling to the login 
validation function, wrapping logic in try-catch blocks and adding 
specific error handling for password strength checks.

Impact & Risks:
- Low risk: Defensive programming improves resilience
- No breaking changes to API

Testing Recommendations:
- Test with malformed inputs (null, undefined, special characters)
- Verify error logs capture exception details
- Ensure valid logins still work correctly
```

---

## Configuration

### Supported Jira Ticket Formats

- `GPTEINFRA-1234`
- `RHCLOUD-567`
- `RHDP-890`
- Any `[A-Z]+-[0-9]+` pattern

### MAAS API Configuration

- **Endpoint:** `https://maas-rhdp.apps.maas.redhatworkshops.io/v1/chat/completions`
- **Model:** `claude-opus-4-6`
- **Format:** OpenAI-compatible API
- **Max Tokens:** 1000 (configurable)
- **Authentication:** Bearer token

### Jira API Configuration

- **Endpoint:** `https://redhat.atlassian.net/rest/api/3`
- **Authentication:** Basic Auth (email + API token)
- **Format:** ADF (Atlassian Document Format)
- **Permissions Required:** Comment on issues

---

## Jira Ticket Permissions

### Security Levels

The bot account can only access tickets without security restrictions or with appropriate permissions.

**Examples:**

| Ticket | Security Level | Bot Access |
|--------|---------------|------------|
| GPTEINFRA-16598 | None (public) | ✅ Yes |
| GPTEINFRA-17017 | "Red Hat Employee" | ❌ No (unless granted) |

**To grant bot access:**

1. Remove security level from ticket, OR
2. Add `rhdp-jira-bot@redhat.com` to project with appropriate role

---

## Troubleshooting

### Validation Fails with "No Jira ticket found"

**Cause:** PR description doesn't contain the required format

**Solution:**
1. Edit PR description
2. Add line: `**Jira:** GPTEINFRA-1234`
3. Save - validation will re-run automatically

### AI Analysis Fails with "Error parsing JSON"

**Cause:** AI response contains unescaped special characters

**Solution:** This should be handled automatically by `jq` escaping. If it persists:
- Check MAAS API response format
- Verify `jq` is installed in runner

### Jira Update Fails with HTTP 404

**Cause:** Ticket doesn't exist or bot doesn't have permission

**Solution:**
1. Verify ticket exists: `https://redhat.atlassian.net/browse/TICKET-123`
2. Check security level - should be "None" or bot should be granted access
3. Verify bot email/token are correct

### Jira Update Fails with HTTP 401

**Cause:** Invalid authentication credentials

**Solution:**
1. Verify `PH_JIRA_API_USER` is correct email
2. Regenerate `PH_JIRA_API_TOKEN` if expired
3. Update organization secrets

### Empty Diff Sent to AI

**Cause:** Using branch names instead of commit SHAs (fixed in v1.0)

**Solution:** Ensure using latest workflow version with `base.sha` and `head.sha`

---

## Error Message Examples

### Clear Error Formatting

All workflows use formatted error messages with:
- ═══ box borders
- ✅/❌ status indicators
- 📝 action items with links
- HTTP codes and API responses

Example:
```
════════════════════════════════════════════════
❌ MAAS API FAILED
════════════════════════════════════════════════

HTTP Status Code: 400

Response:
{"error": {"message": "Invalid request format"}}

════════════════════════════════════════════════
```

---

## Maintenance

### Updating the Workflow

Changes to `rhpds-workflows` repository automatically apply to all consuming repos.

**Deployment:**
1. Make changes to workflows in `rhpds-workflows`
2. Commit and push to `main` branch
3. All repos using `@main` reference get updates immediately

**Testing:**
1. Create test PR in `jira-ai-demo` repository
2. Verify validation passes
3. Merge PR
4. Check Jira ticket for comment

### Monitoring

**Success Indicators:**
- Validation passes on PRs with tickets
- AI analysis completes in ~10-20 seconds
- Jira comments appear within 30 seconds of merge

**Metrics to Track:**
- Validation failure rate
- Average analysis time
- Jira API error rate

---

## Future Enhancements

### Planned Features

1. **Multi-ticket Support**
   - Allow multiple Jira tickets per PR
   - Post analysis to all linked tickets

2. **Custom AI Prompts**
   - Per-repository prompt customization
   - Domain-specific analysis (security, performance, etc.)

3. **Jira Field Updates**
   - Update status (e.g., "In Review" → "Done")
   - Set fix version based on target branch
   - Add labels based on PR changes

4. **Enhanced Diff Analysis**
   - File-by-file breakdown
   - Impact analysis per component
   - Risk scoring

5. **PR Comments**
   - Post AI summary as PR comment in addition to Jira
   - Inline code suggestions

---

## Technical Details

### Reusable Workflow Pattern

**Central Repository:** `rhpds/rhpds-workflows`
- Contains workflow definitions
- Single source of truth
- Version controlled

**Consuming Repositories:**
- Reference workflows via `uses: rhpds/rhpds-workflows/.github/workflows/name.yml@main`
- Pass secrets via `secrets:` block
- No workflow logic duplication

**Benefits:**
- One place to update workflows
- Consistent behavior across repos
- Easier maintenance

### Commit SHA vs Branch Names

**Why SHAs:**
- Branches update after merge → empty diff
- SHAs are immutable → consistent diff
- `base.sha` = state before PR changes
- `head.sha` = state after PR changes

**Formula:**
```bash
git diff base.sha...head.sha
```

This captures exactly what changed in the PR, regardless of merge status.

---

## Repository Structure

```
rhpds-workflows/
├── .github/
│   └── workflows/
│       ├── jira-ai-update.yml       # AI analysis + Jira update
│       └── validate-jira-ticket.yml # Validation check
├── README.md                         # User guide
└── SPEC.md                          # This document

jira-ai-demo/ (example)
├── .github/
│   ├── workflows/
│   │   └── jira-update.yml          # Calls reusable workflows
│   └── pull_request_template.md     # PR template
└── app.py                            # Demo application
```

---

## Security Considerations

### Secrets Management

- **Never commit secrets** to repository
- Use GitHub organization-level secrets
- Rotate API tokens regularly
- Limit bot account permissions to minimum required

### Data Privacy

- AI receives code diffs (may contain sensitive logic)
- Jira comments are visible to project members
- Ensure MAAS endpoint is internal/trusted

### Access Control

- Bot account should only comment, not edit issues
- Validation can be bypassed by admins (by design)
- Consider restricting merge permissions

---

## Support

**Issues:** https://github.com/rhpds/rhpds-workflows/issues  
**Maintainer:** RHDP Team  
**Contact:** rhdp-jira-bot@redhat.com

---

## Changelog

### v1.0 (2026-06-25)
- Initial release
- Validation workflow with clear error messages
- AI analysis with Claude Opus 4.6
- Jira integration with ADF formatting
- PR author attribution in comments
- SHA-based diff extraction (fixes empty diff bug)

---

## License

Internal Red Hat tooling - not for external distribution.
