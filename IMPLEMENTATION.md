# GitHub Bot Implementation Guide

**Version:** 2.0 - Customized Configuration  
**Target System:** Self-hosted Windmill + Claude + GitHub App  
**Deployment:** Docker Compose on your infrastructure  

---

## Executive Summary

This document specifies a complete implementation of an automated development workflow system where:
- **User Interface:** GitHub (issues, PRs, comments, reactions)
- **Orchestration:** Windmill (self-hosted via Docker Compose)
- **AI Engine:** Claude API (analysis, implementation, fixes)
- **Authentication:** GitHub App (dedicated bot identity)
- **Code Repository:** Git/GitHub (version control, CI/CD)

**User Experience:**
1. User creates GitHub issue with optional tags (`bot:simple`, `bot:complex`)
2. Bot analyzes, creates analysis branch with `ANALYSIS.md`
3. Bot comments with proposed actions, waits for approval (👍 reaction)
4. Bot creates action branches and PRs for each action
5. GitHub Actions runs validation (type check, tests)
6. Bot auto-fixes CI failures (behavior based on issue tags)
7. Bot responds to PR review comments
8. Bot auto-merges when checks pass and PR is approved

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                         GITHUB                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Issues  │  │   PRs    │  │ Comments │  │  Actions │   │
│  │  + Tags  │  │          │  │          │  │          │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │             │              │             │          │
│       └─────────────┴──────────────┴─────────────┘          │
│                          │                                   │
│                     Webhooks                                 │
└──────────────────────────┼──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                 WINDMILL (Self-Hosted)                      │
│                    Docker Compose                           │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Workflow Orchestrator                  │    │
│  │                                                     │    │
│  │  Tag-Aware Strategy:                               │    │
│  │  • bot:simple    → Auto-fix immediately            │    │
│  │  • bot:complex   → Comment before fixing           │    │
│  │  • (no tag)      → Default to immediate            │    │
│  │                                                     │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐         │    │
│  │  │ Analyze  │  │ Execute  │  │ Fix CI   │         │    │
│  │  │  Issue   │  │ Actions  │  │ Failures │         │    │
│  │  └──────────┘  └──────────┘  └──────────┘         │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐         │    │
│  │  │  Handle  │  │  Auto    │  │  State   │         │    │
│  │  │ Reviews  │  │  Merge   │  │ Manager  │         │    │
│  │  └──────────┘  └──────────┘  └──────────┘         │    │
│  └────────────────────────────────────────────────────┘    │
│                          │                                   │
└──────────────────────────┼──────────────────────────────────┘
                           │
         ┌─────────────────┴─────────────────┐
         │                                   │
         ▼                                   ▼
┌─────────────────┐              ┌─────────────────┐
│   CLAUDE API    │              │   GITHUB APP    │
│  - Analysis     │              │  - Clone repo   │
│  - Code gen     │              │  - Create PR    │
│  - Fix errors   │              │  - Comment      │
└─────────────────┘              │  - JWT auth     │
                                 └─────────────────┘
```

---

## Configuration Decisions

### ✅ Decision 1: Windmill Deployment
**Selected:** Self-Hosted (Docker Compose)

**Rationale:**
- Full control over infrastructure
- No per-user costs
- Data stays private
- Can scale resources as needed

**Setup Time:** 1-2 hours initial setup
**See:** `docker-compose.yml` artifact for deployment configuration

---

### ✅ Decision 2: GitHub Authentication
**Selected:** GitHub App

**Rationale:**
- Bot appears as dedicated account (not tied to user)
- Fine-grained, auditable permissions
- Works across multiple repos in org
- Higher API rate limits
- Professional appearance

**Required Permissions:**
- Contents: Read & write
- Pull requests: Read & write
- Issues: Read & write
- Workflows: Read & write
- Metadata: Read-only

**Setup Time:** 30 minutes
**See:** Section 5.3 for detailed GitHub App creation steps

---

### ✅ Decision 3: Branch Strategy
**Selected:** Flexible (Support Both Strategies)

You can choose per-issue by using tags:
- **`bot:flat`** → Use flat structure (`bot/{issue-number}-{action-name}`)
- **`bot:hierarchical`** → Use hierarchical (`analysis/`, `action/` prefixes)
- **(no tag)** → Default to hierarchical (recommended)

**Hierarchical Structure (Default):**
```
analysis/{issue-number}-{slug}     # ANALYSIS.md only
action/{issue-number}-{action-1}   # First implementation
action/{issue-number}-{action-2}   # Second implementation
rework/{issue-number}-{iteration}  # If rework needed
```

**Flat Structure:**
```
bot/{issue-number}-{action-1}
bot/{issue-number}-{action-2}
```

---

### ✅ Decision 4: CI Failure Handling
**Selected:** Tag-Based Strategy

Behavior is controlled by issue tags:

| Issue Tag | CI Failure Behavior | Use Case |
|-----------|---------------------|----------|
| `bot:simple` | Immediate auto-fix (max 3 attempts) | Type errors, imports, simple bugs |
| `bot:complex` | Comment before fixing, wait for approval | Logic errors, test failures |
| **(no tag)** | Immediate auto-fix (default) | General development |

**How It Works:**
1. CI fails on PR
2. Bot checks parent issue for tags
3. If `bot:complex`: Comment proposed fix, wait for 👍
4. If `bot:simple` or no tag: Push fix immediately
5. After 3 failed attempts: Always escalate (comment + tag maintainer)

**Configuration:**
```typescript
// In workflow state
interface IssueState {
  // ...
  ci_strategy: "immediate" | "approval_required"
  // Derived from issue tags at analysis time
}
```

---

## Implementation Phases

### Phase 0: Infrastructure Setup (Days 1-2)

**Objective:** Get Windmill running and accessible

**Tasks:**
1. ✅ Deploy Windmill via Docker Compose
2. ✅ Configure reverse proxy (nginx/Caddy) with SSL
3. ✅ Set up PostgreSQL persistence
4. ✅ Create Windmill workspace
5. ✅ Verify web UI accessible

**Deliverables:**
- Windmill accessible at `https://windmill.yourdomain.com`
- Admin account created
- Workspace initialized

**See Artifacts:**
- `docker-compose.yml` - Windmill deployment
- `nginx.conf` - Reverse proxy configuration

---

### Phase 1: GitHub App Setup (Day 3)

**Objective:** Create and configure GitHub App for authentication

**Tasks:**
1. ✅ Create GitHub App in organization settings
2. ✅ Configure permissions (contents, PRs, issues, workflows)
3. ✅ Generate private key
4. ✅ Install app to target repository
5. ✅ Store credentials in Windmill

**Deliverables:**
- GitHub App installed and authorized
- App ID, Installation ID, Private Key stored in Windmill resources
- Webhook URL configured

**GitHub App Configuration:**
```
Name: [Your Bot Name]
Homepage URL: https://windmill.yourdomain.com
Webhook URL: https://windmill.yourdomain.com/api/w/{workspace}/webhooks/github
Webhook secret: [generate random string]

Permissions:
- Repository permissions:
  - Contents: Read & write
  - Pull requests: Read & write  
  - Issues: Read & write
  - Workflows: Read & write
  - Metadata: Read-only

Subscribe to events:
- Issues
- Issue comments
- Pull requests
- Pull request reviews
- Pull request review comments
- Check runs
- Check suites
```

---

### Phase 2: Core Workflow - Analysis (Days 4-5)

**Objective:** Implement issue analysis workflow

**Tasks:**
1. ✅ Create webhook endpoint in Windmill
2. ✅ Implement `analyze-issue` workflow
3. ✅ Add GitHub App JWT authentication
4. ✅ Implement tag detection for strategy selection
5. ✅ Test with sample issue

**Workflow Steps:**
1. Parse webhook (issue.opened)
2. Detect issue tags → set CI strategy
3. Authenticate via GitHub App
4. Clone repository
5. Read project files
6. Call Claude API for analysis
7. Parse actions from analysis
8. Create analysis branch
9. Commit ANALYSIS.md
10. Comment on issue with proposed actions
11. Save state (including CI strategy)

**See Artifact:** `analyze-issue-workflow.ts`

---

### Phase 3: Action Execution (Days 6-8)

**Objective:** Implement action execution after approval

**Tasks:**
1. ✅ Implement approval detection (👍 reaction)
2. ✅ Create `execute-actions` workflow
3. ✅ Implement branch strategy selection (hierarchical vs flat)
4. ✅ For each action:
   - Create appropriate branch
   - Call Claude for implementation
   - Create PR
   - Link to parent issue
5. ✅ Test multi-action execution

**See Artifact:** `execute-actions-workflow.ts`

---

### Phase 4: CI Integration (Days 9-10)

**Objective:** Handle CI failures with tag-aware strategy

**Tasks:**
1. ✅ Create `handle-ci-failure` workflow
2. ✅ Implement tag-based decision logic
3. ✅ For `bot:simple` or no tag:
   - Auto-fix immediately
   - Max 3 attempts
4. ✅ For `bot:complex`:
   - Comment proposed fix
   - Wait for approval
   - Apply fix after 👍
5. ✅ Test both strategies

**See Artifact:** `handle-ci-failure-workflow.ts`

---

### Phase 5: Review & Merge (Days 11-12)

**Objective:** Handle review feedback and auto-merge

**Tasks:**
1. ✅ Implement `handle-review` workflow
2. ✅ Implement `auto-merge` workflow
3. ✅ Test review response
4. ✅ Test auto-merge on approval

**Workflows:**
- Review comment detection
- Claude-powered response generation
- Merge when: CI passed + approved + no conflicts

---

### Phase 6: Polish & Deploy (Days 13-14)

**Objective:** Production-ready system

**Tasks:**
1. ✅ Add comprehensive error handling
2. ✅ Set up monitoring/alerting
3. ✅ Write user documentation
4. ✅ Train team on bot usage
5. ✅ Deploy to production repository

---

## State Management Schema

Windmill provides built-in state via `wmill.setState()` / `wmill.getState()`.

**State Keys Pattern:** `{entity}-{id}`

```typescript
// Issue State: issue-42
interface IssueState {
  issue_number: number
  repo_owner: string
  repo_name: string
  title: string
  
  // Strategy determined from tags
  ci_strategy: "immediate" | "approval_required"
  branch_strategy: "flat" | "hierarchical"
  
  // Workflow state
  analysis_branch: string
  actions: Array<{
    name: string
    description: string
  }>
  current_action_index: number
  status: "analyzing" | "awaiting_approval" | "executing" | "complete" | "failed"
  
  // Timestamps
  created_at: string
  updated_at: string
}

// PR State: pr-156
interface PRState {
  pr_number: number
  issue_number: number
  action_index: number
  action_name: string
  branch_name: string
  
  // CI handling
  ci_strategy: "immediate" | "approval_required"  // Inherited from issue
  status: "pending_ci" | "ci_passed" | "ci_failed" | "approved" | "merged"
  fix_attempts: number
  
  // Timestamps
  created_at: string
  last_fix_at?: string
}
```

**State Access Examples:**
```typescript
// Save issue state
await wmill.setState(`issue-${issue_number}`, {
  issue_number,
  repo_owner,
  repo_name,
  ci_strategy: tags.includes("bot:complex") ? "approval_required" : "immediate",
  branch_strategy: tags.includes("bot:flat") ? "flat" : "hierarchical",
  // ... rest of state
})

// Load issue state
const issueState = await wmill.getState(`issue-${issue_number}`)

// Update PR state
await wmill.setState(`pr-${pr_number}`, {
  ...existingState,
  fix_attempts: existingState.fix_attempts + 1,
  status: "pending_ci"
})
```

---

## Tag System

### Issue Tags

Users can add these tags to issues to control bot behavior:

| Tag | Effect | Description |
|-----|--------|-------------|
| `bot:simple` | CI auto-fix immediately | For simple tasks, auto-fix type errors without asking |
| `bot:complex` | CI requires approval | For complex tasks, comment proposed fixes for review |
| `bot:flat` | Use flat branch structure | All branches at root level: `bot/{issue}-{action}` |
| `bot:hierarchical` | Use hierarchical branches | Organized: `analysis/`, `action/`, `rework/` |
| `bot:skip` | Don't process this issue | Manual development only |

### Default Behavior (No Tags)

If no tags present:
- **CI Strategy:** Immediate auto-fix (same as `bot:simple`)
- **Branch Strategy:** Hierarchical (recommended)

### Tag Detection

```typescript
function detectStrategy(issue: { labels: Array<{ name: string }> }) {
  const tagNames = issue.labels.map(l => l.name.toLowerCase())
  
  return {
    ci_strategy: tagNames.includes("bot:complex") 
      ? "approval_required" 
      : "immediate",
    
    branch_strategy: tagNames.includes("bot:flat")
      ? "flat"
      : "hierarchical",
    
    should_skip: tagNames.includes("bot:skip")
  }
}
```

---

## GitHub App Authentication

### How It Works

GitHub Apps use JWT-based authentication:
1. Generate JWT signed with private key
2. Exchange JWT for installation access token
3. Use access token for API calls (valid 1 hour)

### Implementation

```typescript
import { create } from "npm:djwt"
import { Octokit } from "npm:@octokit/rest"

async function getGitHubAppToken(
  appId: string,
  privateKey: string,
  installationId: string
): Promise<string> {
  // 1. Create JWT
  const now = Math.floor(Date.now() / 1000)
  const payload = {
    iat: now - 60,        // Issued 60 seconds ago
    exp: now + 600,       // Expires in 10 minutes
    iss: appId
  }
  
  // Import private key
  const key = await crypto.subtle.importKey(
    "pkcs8",
    pemToArrayBuffer(privateKey),
    { name: "RSASSA-PKCS1-v1_5", hash: "SHA-256" },
    false,
    ["sign"]
  )
  
  const jwt = await create({ alg: "RS256", typ: "JWT" }, payload, key)
  
  // 2. Get installation token
  const response = await fetch(
    `https://api.github.com/app/installations/${installationId}/access_tokens`,
    {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${jwt}`,
        "Accept": "application/vnd.github+json"
      }
    }
  )
  
  const data = await response.json()
  return data.token  // Valid for 1 hour
}

// Helper: Convert PEM to ArrayBuffer
function pemToArrayBuffer(pem: string): ArrayBuffer {
  const b64 = pem
    .replace(/-----BEGIN PRIVATE KEY-----/, "")
    .replace(/-----END PRIVATE KEY-----/, "")
    .replace(/\s/g, "")
  
  const binary = atob(b64)
  const bytes = new Uint8Array(binary.length)
  for (let i = 0; i < binary.length; i++) {
    bytes[i] = binary.charCodeAt(i)
  }
  return bytes.buffer
}
```

### Windmill Resources

Store these in Windmill as secret resources:

```typescript
// Resource: github_app_id
"123456"

// Resource: github_app_private_key
"-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQC...
-----END PRIVATE KEY-----"

// Resource: github_installation_id
"12345678"

// Resource: webhook_secret
"your-random-webhook-secret-here"
```

---

## Testing Strategy

### Unit Tests

Test individual workflow steps in isolation:

```bash
# Run in Windmill CLI or local Deno
deno test tests/workflows/
```

**Example Test:**
```typescript
Deno.test("Tag detection - complex strategy", () => {
  const issue = {
    labels: [
      { name: "bug" },
      { name: "bot:complex" }
    ]
  }
  
  const strategy = detectStrategy(issue)
  assertEquals(strategy.ci_strategy, "approval_required")
  assertEquals(strategy.branch_strategy, "hierarchical")
})
```

### Integration Tests

Test complete flows end-to-end:

**Test Cases:**
1. ✅ Issue with `bot:simple` → auto-fixes CI failure immediately
2. ✅ Issue with `bot:complex` → comments before fixing
3. ✅ Issue with `bot:flat` → creates flat branch structure
4. ✅ Issue with no tags → uses default behavior
5. ✅ Issue with `bot:skip` → ignored by bot

### Manual Testing Checklist

```markdown
## Pre-Production Testing

### Setup Verification
- [ ] Windmill accessible via HTTPS
- [ ] GitHub App installed to test repo
- [ ] Webhooks receiving events (check deliveries)
- [ ] Resources configured correctly

### Tag-Based Behavior
- [ ] Create issue with `bot:simple` tag
  - [ ] Verify analysis created
  - [ ] Introduce type error in PR
  - [ ] Verify immediate auto-fix (no comment asking permission)
  
- [ ] Create issue with `bot:complex` tag
  - [ ] Verify analysis created
  - [ ] Introduce test failure in PR
  - [ ] Verify bot comments proposed fix
  - [ ] React with 👍
  - [ ] Verify fix applied after approval

- [ ] Create issue with `bot:flat` tag
  - [ ] Verify branches use flat structure: `bot/123-action-name`

- [ ] Create issue with `bot:hierarchical` tag
  - [ ] Verify branches use structure: `analysis/123-*`, `action/123-*`

- [ ] Create issue with no tags
  - [ ] Verify default behavior (immediate + hierarchical)

- [ ] Create issue with `bot:skip` tag
  - [ ] Verify bot ignores it

### Error Cases
- [ ] Invalid webhook signature → rejected (403)
- [ ] GitHub App token expiry → refreshes automatically
- [ ] Max fix attempts (3) → escalates to human
- [ ] Claude API timeout → retries with backoff
```

---

## Monitoring & Alerting

### Metrics to Track

**Windmill Dashboard:**
- Workflow success/failure rate
- Average execution time per workflow
- API call volumes (GitHub, Claude)
- Error frequency by type

**GitHub App:**
- API rate limit usage
- Token refresh frequency
- Webhook delivery success rate

### Alerting Setup

**Critical Alerts** (immediate notification):
- Windmill container stopped
- PostgreSQL connection lost
- GitHub App authentication failed
- Webhook endpoint unreachable

**Warning Alerts** (review within 24h):
- High workflow failure rate (>20%)
- Claude API errors increasing
- GitHub rate limit approaching (>80%)
- Disk space low on Windmill host

**Implementation:**
Use Windmill's built-in error handlers + external monitoring:

```typescript
// In each workflow
error_handler: {
  type: "script",
  path: "shared/error-handler",
  args: {
    severity: "critical",  // or "warning"
    notify_slack: true,
    notify_email: true
  }
}
```

---

## Security Considerations

### Secrets Management

**Never commit:**
- ❌ GitHub App private key
- ❌ Webhook secrets
- ❌ Claude API keys

**Always use Windmill resources:**
- ✅ Store as secret resources (encrypted at rest)
- ✅ Access via `wmill.getResource()`
- ✅ Rotate keys regularly (every 90 days)

### Webhook Verification

**Always verify webhook signatures:**

```typescript
import { createHmac } from "node:crypto"

function verifyWebhookSignature(
  payload: string,
  signature: string,
  secret: string
): boolean {
  const hmac = createHmac("sha256", secret)
  hmac.update(payload)
  const digest = "sha256=" + hmac.digest("hex")
  
  return signature === digest
}

// In webhook handler
export async function handleWebhook(
  body: string,
  headers: Record<string, string>,
  webhook_secret: string
) {
  const signature = headers["x-hub-signature-256"]
  
  if (!verifyWebhookSignature(body, signature, webhook_secret)) {
    throw new Error("Invalid webhook signature")
  }
  
  // Process webhook...
}
```

### GitHub App Permissions

**Principle of least privilege:**
- Only request permissions actually needed
- Use read-only where possible
- Review permissions quarterly

**Current Required Permissions:**
- Contents: Read & write (for creating branches, commits)
- Pull requests: Read & write (for creating/merging PRs)
- Issues: Read & write (for comments, labels)
- Workflows: Read & write (for re-running checks)
- Metadata: Read-only (for repo info)

---

## Deployment Checklist

### Pre-Deployment

- [ ] **Decision Trees Completed**
  - [x] Windmill deployment: Self-hosted Docker
  - [x] GitHub auth: GitHub App
  - [x] Branch strategy: Flexible (tag-based)
  - [x] CI handling: Tag-based (simple/complex)
  - [x] State management: Windmill built-in
  - [x] Error notifications: GitHub comments + Slack

- [ ] **Infrastructure Ready**
  - [ ] Server provisioned (min 2 CPU, 4GB RAM, 20GB disk)
  - [ ] Domain/subdomain configured
  - [ ] SSL certificate obtained
  - [ ] Firewall rules configured (ports 80, 443)

### Deployment Steps

1. **Deploy Windmill**
   - [ ] Copy `docker-compose.yml` to server
   - [ ] Copy `nginx.conf` to server
   - [ ] Run `docker-compose up -d`
   - [ ] Verify accessible at `https://windmill.yourdomain.com`
   - [ ] Create admin account
   - [ ] Create workspace

2. **Create GitHub App**
   - [ ] Go to Organization Settings → Developer → GitHub Apps
   - [ ] Click "New GitHub App"
   - [ ] Configure permissions (see Phase 1)
   - [ ] Generate and download private key
   - [ ] Install app to target repository
   - [ ] Note App ID and Installation ID

3. **Configure Windmill Resources**
   - [ ] Add `github_app_id`
   - [ ] Add `github_app_private_key` (paste entire PEM)
   - [ ] Add `github_installation_id`
   - [ ] Add `webhook_secret` (generate random string)
   - [ ] Add `anthropic_key`
   - [ ] Add `slack_webhook` (optional)

4. **Deploy Workflows**
   - [ ] Upload all workflow scripts to Windmill
   - [ ] Create flows via UI or YAML
   - [ ] Configure webhook endpoint
   - [ ] Test webhook delivery from GitHub

5. **Configure GitHub Webhook**
   - [ ] Repository Settings → Webhooks → Add
   - [ ] URL: `https://windmill.yourdomain.com/api/w/{workspace}/webhooks/github`
   - [ ] Content type: `application/json`
   - [ ] Secret: (matches `webhook_secret` resource)
   - [ ] Events: Issues, PRs, Comments, Checks
   - [ ] Verify delivery successful (green ✓)

6. **Add GitHub Actions**
   - [ ] Create `.github/workflows/validation.yml`
   - [ ] Configure to run on PRs to `analysis/**`, `action/**`
   - [ ] Test triggers properly

7. **Testing**
   - [ ] Run manual test checklist
   - [ ] Create test issue with each tag combination
   - [ ] Verify end-to-end flows work
   - [ ] Check logs for errors

8. **Go Live**
   - [ ] Update documentation
   - [ ] Announce to team
   - [ ] Monitor first few issues closely

---

## Troubleshooting

### GitHub App Authentication Failures

**Symptom:** "Bad credentials" or 401 errors

**Diagnosis:**
```bash
# Check if private key is valid
openssl rsa -in private-key.pem -check

# Test JWT generation
deno run --allow-all test-github-app-auth.ts
```

**Solutions:**
- Verify App ID is correct (check GitHub App settings)
- Verify Installation ID is correct (check installed apps in repo)
- Ensure private key is complete (includes BEGIN/END markers)
- Check JWT expiration (should be <10 minutes)
- Verify app is installed to the target repository

### Webhook Not Triggering Workflows

**Symptom:** Issue created but no bot response

**Diagnosis:**
1. GitHub Repo → Settings → Webhooks → Recent Deliveries
2. Look for red X (failed) or yellow ⚠ (timeout)
3. Click delivery to see request/response

**Solutions:**
- **403 Forbidden:** Signature verification failed
  - Ensure `webhook_secret` matches in GitHub and Windmill
- **404 Not Found:** Wrong webhook URL
  - Verify format: `https://windmill.yourdomain.com/api/w/{workspace}/webhooks/github`
  - Check workspace name is correct
- **500 Server Error:** Workflow error
  - Check Windmill logs: `docker-compose logs -f windmill`
  - Look for stack traces
- **Timeout:** Workflow too slow
  - Return 202 immediately, process async
  - Add timeout to long-running steps

### Tag Detection Not Working

**Symptom:** Bot ignores `bot:complex` tag

**Diagnosis:**
```typescript
// Add logging in tag detection function
console.log("Issue labels:", issue.labels.map(l => l.name))
console.log("Detected strategy:", strategy)
```

**Solutions:**
- Verify tag name matches exactly (case-insensitive comparison)
- Check tag was added before webhook fired (GitHub may not include newly added tags in `issue.opened` event)
- Workaround: Re-fetch issue details in workflow to get latest tags

### Docker Compose Issues

**Symptom:** Containers won't start

**Solutions:**
- Check logs: `docker-compose logs windmill`
- Verify ports not in use: `netstat -tulpn | grep :8000`
- Check disk space: `df -h`
- Verify PostgreSQL started: `docker-compose ps`
- Reset: `docker-compose down -v && docker-compose up -d`

---

## Next Steps

1. **Review this guide** and ensure you understand the architecture
2. **Set up infrastructure** (Phase 0: Windmill deployment)
3. **Create GitHub App** (Phase 1: Authentication)
4. **Implement workflows** incrementally:
   - Start with analysis workflow
   - Test thoroughly before moving on
   - Add execution workflow
   - Add CI handling
   - Add review/merge workflows
5. **Test with real issues** (start with `bot:simple` tasks)
6. **Monitor and iterate** based on team feedback

**Key Principle:** Build incrementally, test each workflow thoroughly before proceeding.

---

## Appendix A: File Structure

```
project-root/
├── windmill/
│   ├── docker-compose.yml              # Windmill deployment
│   ├── nginx.conf                      # Reverse proxy config
│   ├── flows/
│   │   ├── analyze-issue.flow.ts       # Main analysis workflow
│   │   ├── execute-actions.flow.ts     # Action execution
│   │   ├── handle-ci-failure.flow.ts   # Auto-fix CI failures
│   │   ├── handle-review.flow.ts       # Respond to reviews
│   │   └── auto-merge.flow.ts          # Merge approved PRs
│   ├── scripts/
│   │   ├── github-app-auth.ts          # GitHub App JWT authentication
│   │   ├── git-helpers.ts              # Git operations
│   │   ├── claude-api.ts               # Claude API wrappers
│   │   ├── parsers.ts                  # Parse markdown, code
│   │   └── tag-detector.ts             # Detect and parse issue tags
│   └── resources/
│       ├── github_app_id
│       ├── github_app_private_key
│       ├── github_installation_id
│       ├── webhook_secret
│       └── anthropic_key
│
├── .github/
│   └── workflows/
│       └── validation.yml              # CI checks
│
├── tests/
│   ├── workflows/
│   │   ├── analyze-issue.test.ts
│   │   └── tag-detection.test.ts
│   └── e2e/
│       └── full-workflow.test.ts
│
└── docs/
    ├── USER_GUIDE.md                   # How to use the bot
    ├── ADMIN_GUIDE.md                  # Deployment & config
    └── TROUBLESHOOTING.md              # Common issues

```

---

## Appendix B: Quick Reference

### Issue Tags Quick Reference

```
bot:simple      → CI auto-fixes immediately (max 3 attempts)
bot:complex     → CI comments before fixing, waits for approval
bot:flat        → Use flat branch structure
bot:hierarchical → Use analysis/action/rework structure
bot:skip        → Don't process this issue
```

### Windmill State Keys

```
issue-{number}  → IssueState (analysis, actions, strategy)
pr-{number}     → PRState (CI status, fix attempts)
```

### Common Commands

```bash
# Windmill
docker-compose logs -f windmill         # View logs
docker-compose restart windmill         # Restart
docker-compose down -v                  # Full reset

# GitHub App
# Generate new private key: GitHub App settings → Generate private key

# Webhook Testing
curl -X POST https://windmill.yourdomain.com/api/w/workspace/webhooks/github \
  -H "Content-Type: application/json" \
  -H "X-GitHub-Event: ping" \
  -d '{"zen": "test"}'
```

---

**END OF IMPLEMENTATION GUIDE**
