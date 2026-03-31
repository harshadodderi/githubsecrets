# You Made a Private Repo Public by Mistake — Now What? A Complete GitHub Secrets Guide

> **🚨 Already in an incident?** Skip to [Incident Response](#incident-response) right now. Come back and read the rest after you've rotated everything.

---

It's 11:43 PM. A developer on your team is cleaning up repositories — archiving old projects, updating descriptions. They toggle one repo to public by mistake. They notice thirty seconds later and toggle it back.

It's already too late.

Within seconds of a repository becoming public, automated bots monitoring GitHub's public event stream have cloned it, scanned every file and every commit in history, and extracted any credential-shaped strings they find. By the time your developer refreshed the page, that AWS key was being tested against the AWS API. By midnight, crypto miners were running in your account.

This isn't a hypothetical. It's the exact pattern behind thousands of real incidents. The scraping infrastructure is always running, always watching, and faster than any human reaction time.

**This guide exists so you're prepared before it happens, and know exactly what to do if it does.**

---

## Table of Contents

1. [How Secrets Actually Leak — The Full Picture](#how-secrets-leak)
2. [The Defense-in-Depth Model](#defense-in-depth)
3. [Incident Response — What to Do Right Now](#incident-response)
4. [Why Git History Is the Hidden Danger](#git-history-danger)
5. [Removing Secrets from Git History](#removing-secrets-from-git-history)
6. [Secrets Management — The Right Way](#secrets-management)
7. [GitHub Actions Hardening](#github-actions-hardening)
8. [Preventive Controls](#preventive-controls)
9. [Hard Nos — Common Mistakes That Cause Incidents](#hard-nos)
10. [Quick Reference Checklists](#quick-reference-checklist)
11. [References](#references)

---

## How Secrets Actually Leak — The Full Picture {#how-secrets-leak}

Before we talk about fixes, let's be honest about how secrets end up in git in the first place. It's rarely carelessness — it's almost always a workflow gap.

**The most common leak patterns:**

- **`.env` not in `.gitignore`** — A developer creates a `.env` file, adds real credentials, and commits it before anyone adds `.gitignore`. Every subsequent commit now carries that file in history.
- **Hardcoded during "quick testing"** — `API_KEY = "sk-real-key-here"` gets committed "just to test this one thing." The test passes. The cleanup commit never happens.
- **Secrets in GitHub Actions workflow files** — Especially common when migrating from another CI system where secrets were just environment variables in a YAML file.
- **Repository visibility change** — An internal tool repo gets open-sourced. No one audited git history first. Years of credentials are now public.
- **Accidental fork or template creation** — Forking a private repo to a personal account, where the fork is public by default.
- **Third-party integrations** — Some older tools (certain deploy systems, legacy Heroku configs) stored their configuration in git-tracked files.

**What happens after exposure — the timeline:**

| Time after exposure | What has happened |
|---|---|
| < 5 seconds | GitHub's public event stream picks up the repo/push |
| 5–30 seconds | Automated scrapers clone the repo and extract files |
| 30–120 seconds | Credential strings are extracted and tested against the API |
| 2–10 minutes | Valid credentials are being actively used |
| 1–24 hours | Full exploitation — mining, data exfiltration, lateral movement |

This is why "I noticed in 30 seconds and made it private" is not a safety net. You are reacting to an event that the attacker's automation already won.

**Real-world consequences by secret type:**

- **AWS keys** — Immediately used to launch crypto mining EC2 instances. Bills of $50,000+ in under 24 hours are documented cases.
- **Database credentials** — Exfiltration of all customer data, potential ransomware on the DB.
- **GitHub tokens** — Used to clone all private repositories in your organization, exfiltrating your entire codebase.
- **Stripe / payment keys** — Fraudulent charges, refund abuse, or resale of the key on dark web marketplaces.
- **Twilio / SendGrid keys** — Mass spam campaigns billed to your account.
- **SSH keys** — Backdoor access to every server the key is deployed on.

---

## The Defense-in-Depth Model {#defense-in-depth}

Security for secrets isn't a single control — it's overlapping layers. Each layer catches what the previous one misses. Think of it as five gates:

```
[ Your machine ]
      ↓  ← Layer 1: pre-commit hook (gitleaks, detect-secrets)
[ git commit ]
      ↓  ← Layer 2: .gitignore, code review, .env.example pattern
[ git push ]
      ↓  ← Layer 3: GitHub push protection, secret scanning
[ Remote repository ]
      ↓  ← Layer 4: repo visibility policy, branch protection, CI scan
[ Public internet ]
      ↓  ← Layer 5: OIDC, short-lived credentials, least privilege, secrets manager
[ Cloud / services ]
```

The goal is that a secret never reaches the next layer down. But if it does — you have a control there too. No single layer is sufficient. All of them together make a breach very unlikely and, if it happens, the blast radius small.

This guide covers every layer.

---

## Incident Response — What to Do Right Now {#incident-response}

If you are here because an exposure just happened: **do not panic, and do not skip steps**. The order matters.

### Step 1 — Rotate immediately, before anything else

Every exposed credential must be treated as **fully compromised from the moment of exposure**. Do not wait to check logs first. Do not wait to confirm it was scraped. Assume it was, and act accordingly.

**By service:**

- **AWS access keys**: IAM Console → Users → Security credentials → Deactivate the exposed key → Create new key pair → Update all systems using the old key → Delete the old key. Don't just deactivate — delete it.
- **GitHub Personal Access Tokens**: Settings → Developer Settings → Personal Access Tokens → Revoke.
- **GitHub Actions secrets**: Repository Settings → Secrets and variables → Actions → Update the secret value.
- **Database passwords**: Change immediately via your DB console or CLI. Also audit whether the database is publicly network-accessible — if so, restrict it to your application's IP range immediately.
- **Third-party API keys** (Stripe, Twilio, SendGrid, etc.): Each has a "roll key" or "revoke" option in their dashboard. Do it immediately, then update your secrets store with the new value.
- **SSH keys**: Remove the compromised key from `~/.ssh/authorized_keys` on every server it was deployed to. Remove from GitHub under Settings → SSH and GPG keys.

> **The rule:** Rotate first, investigate second. Rotation takes minutes. A breach investigation and remediation takes weeks.

### Step 2 — Set the repository to private

If the repo is still public, make it private immediately. This stops new exposure from ongoing scraping but does not undo existing exposure.

**Log the exact timestamps**: when the repo became public, when you noticed, when you set it private. You will need this for the audit and potentially for compliance reporting.

### Step 3 — Audit access logs for abuse

Now that rotation is done, investigate what happened during the exposure window.

- **AWS CloudTrail**: Filter by the IAM user/role attached to the exposed key. Look for `AssumeRole`, EC2 `RunInstances`, S3 `GetObject`/`PutObject`, Lambda `CreateFunction`, or IAM `CreateUser` — all classic post-compromise actions. Check AWS Cost Explorer for sudden spikes in EC2 or Lambda spend.
- **GCP**: Cloud Audit Logs → filter by the service account.
- **Azure**: Azure Monitor → Activity Log.
- **GitHub audit log**: Settings → Security log — look for OAuth app grants, new SSH keys, or private repo access from unfamiliar IPs.
- **Third-party services**: Check each service's own event or activity log for anomalous API calls during the exposure window.

If you find evidence of abuse, escalate to your security team and preserve logs — they are evidence.

### Step 4 — Purge the secret from git history

See the dedicated section: [Removing Secrets from Git History](#removing-secrets-from-git-history). Do this after rotation — not instead of it.

### Step 5 — Notify and document

- **Internally**: Notify your security team and engineering lead, even if you believe no abuse occurred. Document the full timeline.
- **Compliance**: If any regulated data (PII, PHI, financial records, PCI data) may have been accessible via the exposed credential, check your legal obligations. GDPR, HIPAA, and PCI-DSS all have breach notification requirements with short windows.
- **Affected users**: If there's any chance customer data was accessed, follow your organization's incident communication policy.

### Step 6 — Post-incident hardening

Once the incident is resolved, implement the controls in [Preventive Controls](#preventive-controls). Run a full historical audit on all other repositories in your organization.

---

## Why Git History Is the Hidden Danger {#git-history-danger}

Most developers know committed secrets are bad. What fewer understand is *why* deleting the file and making a new commit doesn't fix anything.

Git stores content using a **content-addressed object store**. When you commit a file containing a secret, git creates:
- A **blob** object: the file's content, hashed and stored
- A **tree** object: a snapshot of the directory at that commit, referencing the blob
- A **commit** object: metadata plus a pointer to the tree

These objects remain in the repository's `.git` directory permanently, even after you delete the file in a later commit. The later commit just creates a new tree that doesn't reference the old blob — but the blob itself is still there.

```bash
# This does NOT remove the secret from history
git rm .env
git commit -m "remove .env"
git push

# The old blob is still reachable:
git log --all --full-history -- .env          # find old commits
git show <old-commit-hash>:.env              # read the file contents
git checkout <old-commit-hash> -- .env        # restore the file
```

Anyone who cloned the repo — including the bots that scraped it — has the full `.git` directory with all these objects. Even after a force-push that rewrites history, GitHub caches objects for a grace period.

**This is why the correct flow is always:**

1. **Rotate the credential** — makes it worthless regardless of who has it
2. **Clean the history** — prevents future accidental re-exposure from old branches or cached objects

In that order. Rotation is the fix. History cleanup is hygiene. Both matter.

---

## Removing Secrets from Git History {#removing-secrets-from-git-history}

### Before you start: take a full backup

```bash
# Create a bare mirror clone before touching anything
git clone --mirror https://github.com/YOUR_ORG/YOUR_REPO.git repo-backup.git
```

Keep this until you've confirmed the rewrite and force-push went cleanly.

### Option A — git-filter-repo (recommended)

`git-filter-repo` is the current standard. It replaced the deprecated `git filter-branch` (now removed from some git distributions). Significantly faster on large repos and handles edge cases correctly.

**Install:**

```bash
# macOS
brew install git-filter-repo

# Cross-platform via pip
pip install git-filter-repo

# Verify
git filter-repo --version
```

**Remove a specific file from all history:**

```bash
# Removes .env from every commit across all branches and tags
git filter-repo --path .env --invert-paths

# File in a subdirectory
git filter-repo --path config/credentials.yml --invert-paths

# Remove multiple files at once
git filter-repo --path .env --path secrets.json --path .aws/credentials --invert-paths
```

**Replace a specific secret string across all history** (useful when the secret appeared in multiple files or commits):

```bash
# Create a replacements file — format: LITERAL_STRING==>REPLACEMENT
cat > replacements.txt << 'EOF'
AKIAIOSFODNN7EXAMPLE==>REDACTED_AWS_KEY_ID
wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY==>REDACTED_AWS_SECRET
ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==>REDACTED_GITHUB_TOKEN
EOF

git filter-repo --replace-text replacements.txt
```

**Re-add the remote and force-push:**

```bash
# git-filter-repo removes the remote as a safety measure — re-add it
git remote add origin https://github.com/YOUR_ORG/YOUR_REPO.git

# Force push all branches and all tags
git push origin --force --all
git push origin --force --tags
```

> ⚠️ **Critical:** All collaborators must delete their local clones and re-clone fresh. Anyone who pushes from an old clone will re-introduce the old history. Communicate this to the entire team before they do any work.

### Option B — BFG Repo Cleaner

BFG is simpler for common cases and requires less setup, though `git-filter-repo` handles more edge cases.

```bash
# Requires Java. Download bfg.jar from https://rtyley.github.io/bfg-repo-cleaner/

# Work on a mirror clone
git clone --mirror https://github.com/YOUR_ORG/YOUR_REPO.git

# Remove a specific file from all history (BFG keeps the current commit intact by default)
java -jar bfg.jar --delete-files .env YOUR_REPO.git

# Replace sensitive strings (one replacement per line in passwords.txt)
java -jar bfg.jar --replace-text passwords.txt YOUR_REPO.git

# Clean up dangling objects and push
cd YOUR_REPO.git
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force
```

### What NOT to do

| Approach | Why it doesn't work |
|---|---|
| `git rm <file>` + new commit | Removes from working tree only. Blob object and all previous commits unchanged. |
| `git commit --amend` | Only rewrites the most recent commit. All earlier commits are untouched. |
| `git rebase -i` and drop the commit | Rewrites linear history but leaves blob objects until you run `git gc`, and doesn't handle all branches. |
| Making the repo private | Stops new exposure. Does not remove from history or undo scraping that already occurred. |
| Deleting and recreating the repo | Works for history, but you lose all issues, PRs, stars, wikis, and other metadata. Also, existing forks retain the full history. |

### After the rewrite: request GitHub cache invalidation

GitHub caches repository objects and may continue to serve old commit SHAs for a period after a force-push. To request a full cache purge, contact GitHub Support at [support.github.com](https://support.github.com), provide the repository name and the specific commits that contained the secret. This is especially important if the repo was briefly public.

---

## Secrets Management — The Right Way {#secrets-management}

### The hierarchy of secrets storage

| Method | Risk level | Notes |
|---|---|---|
| Hardcoded in source code | 🔴 Critical | Never. Full stop. |
| Committed `.env` file | 🔴 Critical | In git history permanently, cloned by everyone |
| Untracked `.env` file | 🟡 Medium | Better, but per-machine risk, no rotation, no audit trail |
| GitHub Secrets | 🟢 Good | Encrypted at rest, masked in logs, scoped to repo/env/org |
| Dedicated secrets manager | 🟢 Best | Vault, AWS Secrets Manager — versioned, audited, auto-rotatable |

### GitHub Secrets — levels and scope

GitHub provides encrypted secrets at three levels:

| Level | Scope | Best for |
|---|---|---|
| Repository | Single repo | Project-specific credentials |
| Environment | Specific deployment environment | Environment-scoped secrets — prod DB only accessible to prod deployments |
| Organization | All repos in the org | Shared credentials like a package registry publish token |

Access in workflows with `${{ secrets.MY_SECRET_NAME }}`. GitHub automatically masks the exact value in logs. **Note:** masking only covers exact matches — base64-encoded or URL-encoded forms of the secret will not be masked. Never log secrets even indirectly.

### Prefer OIDC over static credentials

Static cloud credentials (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`) are a long-lived liability — they don't expire, must be manually rotated, and if leaked, remain valid until you notice. OIDC eliminates this by letting GitHub Actions authenticate directly to your cloud provider as a trusted identity, generating short-lived, per-run credentials automatically.

**AWS OIDC setup — one-time configuration:**

Step 1: Create an OIDC Identity Provider in IAM.
- URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

Step 2: Create an IAM role with this trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

The `sub` condition scopes this role to only your `main` branch. For deploy workflows, this is exactly the right scope.

**If you provision AWS infrastructure with Terraform:**

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

resource "aws_iam_role" "github_actions" {
  name = "github-actions-deploy"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.github.arn }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:sub" = "repo:YOUR_ORG/YOUR_REPO:ref:refs/heads/main"
        }
      }
    }]
  })
}
```

**In your workflow — zero stored credentials:**

```yaml
permissions:
  id-token: write    # Required to request the OIDC token
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
          aws-region: ap-south-1

      - name: Deploy
        run: aws s3 sync ./dist s3://my-bucket
```

No `AWS_ACCESS_KEY_ID` stored anywhere. Credentials are minted per-run and auto-expire.

### Dedicated secrets managers

For production workloads, consider a dedicated secrets manager so applications fetch credentials at runtime — removing the need to store secrets in environment files or CI secrets at all:

- **AWS Secrets Manager** — Native integration with EC2, Lambda, ECS. Automatic rotation for RDS passwords.
- **HashiCorp Vault** — Open source, self-hosted. Supports dynamic secrets (credentials that exist only for the duration of a single request).
- **GCP Secret Manager** — Strong IAM integration with Google Cloud services.
- **Doppler / Infisical** — Developer-friendly, good GitHub Actions integrations, easy to adopt incrementally.

### The .env.example pattern

Every project should have two files:
- `.env.example` — **committed to git**, documents every required variable with placeholder values
- `.env` — **gitignored**, contains real credentials, never committed

```bash
# .env.example — committed to git, safe to share
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
AWS_REGION=us-east-1
STRIPE_SECRET_KEY=sk_test_REPLACE_WITH_YOUR_KEY
SENDGRID_API_KEY=SG.REPLACE_WITH_YOUR_KEY
GITHUB_TOKEN=ghp_REPLACE_WITH_YOUR_TOKEN
```

Add a startup validation in your app that checks all required environment variables are set and fails loudly if they aren't. This prevents the failure mode where a developer runs locally without a required secret and gets a confusing runtime error instead of a clear "missing config" message.

### Secrets hygiene principles

- **Never reuse secrets across environments.** Dev, staging, and prod must have completely separate credentials. A staging breach should never be able to reach prod.
- **Apply least privilege.** A deploy job that only needs to push to S3 should not have IAM permissions to create EC2 instances. Scope every credential to exactly what it needs.
- **Rotate on a schedule.** Don't wait for an incident. Long-lived credentials accumulate risk over time. Set a calendar reminder for quarterly rotation of all long-lived secrets.
- **Never log secrets.** Not in application logs, not in CI output, not in error messages. Logs often end up in S3, Elasticsearch, or third-party SaaS with broader access than your codebase.

---

## GitHub Actions Hardening {#github-actions-hardening}

### Pin third-party actions to a full commit SHA

Tags like `@v3` or `@v4` are mutable — a compromised maintainer account can silently redirect a tag to malicious code. Your workflow runs it on the next push, with full access to your secrets and your cloud environment.

```yaml
# ❌ Unsafe — tag can be silently redirected to malicious code
- uses: actions/checkout@v4

# ✅ Safe — SHA is immutable, cannot be changed
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```

Always add a comment with the human-readable version. To find the SHA: go to the action's GitHub releases, click the commit link next to the version tag, copy the full 40-character SHA.

Use [Dependabot for GitHub Actions](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot) to get automated PRs when new versions are released — you stay updated without manually tracking each action.

### Always define a permissions block

Without an explicit `permissions:` block, `GITHUB_TOKEN` inherits defaults from your repository settings — often broader than any individual workflow needs.

```yaml
# Set a top-level default of deny everything
permissions: {}

jobs:
  build:
    permissions:
      contents: read        # read the repo
      packages: write       # publish to GitHub Packages

  test:
    permissions:
      contents: read        # tests only need to read

  deploy:
    permissions:
      contents: read
      id-token: write       # OIDC token request
```

Setting `permissions: {}` at the workflow level and granting per-job is the strictest approach and the right default for new workflows.

### Never pass secrets to fork-triggered workflows

The `pull_request` trigger correctly isolates forks — they run without secrets and with read-only `GITHUB_TOKEN`. The `pull_request_target` trigger runs with write access and secrets even for fork PRs.

```yaml
# ❌ Dangerous — fork PR code runs with write access and secrets
on:
  pull_request_target:

# ✅ Safe — forks are fully isolated
on:
  pull_request:
```

If you need `pull_request_target` for a legitimate reason (auto-labeling, commenting from bots), never check out and execute PR code in that context. Separate the code execution into a separate `pull_request` job and only use `pull_request_target` for the privileged action that doesn't touch untrusted code.

### Never echo or print secrets

GitHub masks the exact secret value in logs — but only exact matches. Derived values won't be masked.

```yaml
# ❌ Direct echo — even masked, this is bad practice
- run: echo "Token is ${{ secrets.API_TOKEN }}"

# ❌ Base64 form will not be masked
- run: echo ${{ secrets.API_TOKEN }} | base64

# ✅ Always inject secrets via env:, never inline in run:
- name: Call API
  run: curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
```

Never use `${{ secrets.NAME }}` inline inside a `run:` block. Always pass via `env:`.

### Enable branch protection rules

Branch protection prevents direct pushes to `main` and lets you require the secret scanning CI job to pass before any PR can be merged.

Go to **Repository Settings → Branches → Add branch protection rule** for `main`:

- ✅ Require a pull request before merging
- ✅ Require status checks to pass (add your secret scanning job here)
- ✅ Require at least one approving review
- ✅ Do not allow bypassing the above settings

The last setting is important — without it, admins can bypass, which means the protection is only as strong as your weakest admin.

### Rotate secrets after any structural change

Rotate credentials whenever:
- A workflow file is modified
- Repository visibility changes (private ↔ public)
- A contributor with repo access leaves the organization
- An action dependency is updated to a new pinned version
- A CI/CD integration is added or changed

---

## Preventive Controls {#preventive-controls}

### Layer 1 — Pre-commit hooks: catch it before it's staged

Pre-commit hooks are the earliest possible control. They add zero friction for clean commits and stop leaks before the secret is ever recorded in git's object store.

**Install the pre-commit framework:**

```bash
pip install pre-commit    # or: brew install pre-commit
```

**Add `.pre-commit-config.yaml` to your repo root:**

```yaml
repos:
  # gitleaks — high signal-to-noise, covers 100+ secret types
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks

  # detect-secrets — good for custom patterns and false-positive management
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']

  # General hygiene
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-added-large-files    # catches accidentally added credential files
      - id: detect-private-key         # basic PEM/RSA key detection
      - id: check-merge-conflict
```

**Activate for every developer (run once after cloning):**

```bash
pre-commit install
```

Commit `.pre-commit-config.yaml` to the repo. Add `pre-commit install` to your project's setup script or `Makefile` so new team members get it automatically.

**Managing false positives with a baseline:**

```bash
# Scan current codebase and mark known non-secrets as accepted
detect-secrets scan > .secrets.baseline
git add .secrets.baseline
git commit -m "add detect-secrets baseline"
```

### Layer 2 — GitHub push protection: block at the remote

GitHub's native secret scanning covers 200+ secret types across major cloud providers, SaaS services, and developer tools. Push protection blocks the `git push` itself when a known pattern is detected — before the secret lands in the remote.

**Enable for a single repository:**
Repository → Settings → Security → Code security and analysis → Secret scanning → Enable → Push protection → Enable

**Enable org-wide:**
Organization → Settings → Code security and analysis → Secret scanning → Enable for all repositories → Push protection → Enable for all repositories

When push protection fires, the developer sees exactly which file and which line contains the detected secret, with options to remove it or bypass with a documented reason. Bypass events create an audit log entry.

### Layer 3 — CI scanning: the backstop

Even with pre-commit hooks installed, add a scanning job in CI. Some developers work around hooks, some IDEs commit directly, and new developers may not have run `pre-commit install` yet.

```yaml
name: Security checks

on: [push, pull_request]

jobs:
  secret-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
        with:
          fetch-depth: 0    # Full history — scan all commits on the branch, not just the latest

      - name: Scan for secrets
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

`fetch-depth: 0` is important — the default shallow clone misses secrets in commits that aren't the HEAD.

### Layer 4 — Repository and org hygiene

**`.gitignore` for every project:**

```gitignore
# Secrets and credentials
.env
.env.*
!.env.example
*.pem
*.key
*.p12
*.jks
*.pfx
*.asc
credentials.json
service-account*.json
*_rsa
*_rsa.pub
*_ed25519
*_ed25519.pub

# Cloud provider configs
.aws/
.gcloud/
kubeconfig
*.kubeconfig

# IDE configs (can contain paths and tokens)
.idea/
.vscode/settings.json
*.sublime-workspace
*.code-workspace
```

**Organization-level settings:**
- Set new repos to **private by default**: Org settings → Member privileges → Default repository visibility → Private
- Require documented approval to change any repo to public
- Audit all public repositories quarterly

**Branch protection on `main`** (see [GitHub Actions Hardening](#github-actions-hardening) above).

### Layer 5 — Historical audit

Run a full historical scan when you first implement these controls, to catch secrets committed before the controls existed:

```bash
# Scan entire commit history of current repo
gitleaks detect --source . --log-opts="--all"

# Export results as JSON for structured review
gitleaks detect --source . --log-opts="--all" --report-format json --report-path gitleaks-report.json

# TruffleHog — excellent for verified, high-confidence findings only
trufflehog git https://github.com/YOUR_ORG/YOUR_REPO --only-verified

# Scan an entire GitHub organization
trufflehog github --org=YOUR_ORG --only-verified
```

Any findings from the historical audit should be rotated immediately and cleaned from history using `git-filter-repo`.

---

## Hard Nos — Common Mistakes That Cause Incidents {#hard-nos}

| The mistake | Why it's wrong |
|---|---|
| "I'll push this key temporarily and remove it in the next commit" | Bots scan continuously. The moment it hits the remote, assume it's been harvested. There is no "before anyone sees it." |
| "I made it private in 30 seconds so it's fine" | You are reacting to an automated process that operates in under 5 seconds. Treat any public exposure as a confirmed leak regardless of duration. |
| Deleting the file and committing to "remove" the secret | git's object store retains the blob permanently. Deletion only removes the file from the working tree. The secret is still in every previous commit. |
| Reusing a leaked credential after rotation | Generate a fresh credential. The infrastructure that allowed the leak may still be compromised. |
| Using real credentials against public test or staging services | "Test" services often share infrastructure with prod or have their own exploitable value. Use dedicated sandbox credentials with zero prod access. |
| Pinning actions to a mutable tag like `@v4` | Tags are mutable pointers. A compromised maintainer account can silently redirect `@v4` to malicious code. Pin to a SHA. |
| Echoing secrets in workflow steps | GitHub's masking only covers the exact secret value. Base64, URL-encoded, or partial forms appear in logs unmasked. |
| Using `workflow_dispatch` inputs to pass secrets | Inputs are logged in the GitHub Actions run UI, accessible to anyone with repo read access. Use GitHub Secrets instead. |
| Long-lived static cloud credentials when OIDC is available | Static keys don't expire, accumulate over time, and require manual rotation. OIDC credentials are per-run and auto-expire. |
| Committing a `.env` because it "only has dev values" | Dev values often share patterns or services with staging. Once the pattern is established, someone commits a prod `.env` the same way. |

---

## Quick Reference Checklists {#quick-reference-checklist}

### Before merging any PR

- [ ] `gitleaks` pre-commit hook ran clean on all commits
- [ ] `.env` and all credential files are in `.gitignore` and not tracked (`git ls-files | grep .env`)
- [ ] Any new workflow files use `${{ secrets.NAME }}` — no hardcoded values
- [ ] Third-party actions are pinned to full commit SHAs
- [ ] `permissions:` block is defined with minimum required access
- [ ] No `workflow_dispatch` inputs used to pass secrets

### Before making any repository public

- [ ] Full history scan: `gitleaks detect --source . --log-opts="--all"`
- [ ] Confirm no `.env*` files tracked: `git ls-files | grep -E '\.env'`
- [ ] Review all workflow files for hardcoded credentials
- [ ] GitHub Secret Scanning and push protection enabled
- [ ] Third-party action references pinned to SHAs
- [ ] Engineering lead has reviewed the full repository
- [ ] Branch protection configured on `main`

### After an exposure incident

- [ ] All exposed credentials rotated (deleted and replaced — not just deactivated)
- [ ] Sessions and OAuth tokens invalidated
- [ ] Access logs reviewed for the full exposure window
- [ ] Evidence of abuse escalated if found
- [ ] Git history rewritten with `git-filter-repo`
- [ ] All collaborators notified to delete local clones and re-clone
- [ ] GitHub Support contacted to invalidate cache
- [ ] Full incident timeline documented
- [ ] Post-incident controls implemented (push protection, pre-commit hooks, CI scan)
- [ ] All other org repositories audited for similar issues

---

## References {#references}

**GitHub official documentation:**
- [Removing sensitive data from a repository](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)
- [Using secrets in GitHub Actions](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [Secret scanning and push protection](https://docs.github.com/en/code-security/secret-scanning/about-secret-scanning)
- [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [Keeping Actions up to date with Dependabot](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot)

**Tools:**
- [git-filter-repo](https://github.com/newren/git-filter-repo) — history rewriting (recommended)
- [gitleaks](https://github.com/gitleaks/gitleaks) — secret detection
- [detect-secrets](https://github.com/Yelp/detect-secrets) — Yelp's scanner, good for false-positive management
- [truffleHog](https://github.com/trufflesecurity/trufflehog) — deep history scanning with credential verification
- [BFG Repo Cleaner](https://rtyley.github.io/bfg-repo-cleaner/) — simpler alternative for history rewriting
- [pre-commit](https://pre-commit.com/) — pre-commit hook framework

**Standards and further reading:**
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [OWASP Top 10 — A02: Cryptographic Failures](https://owasp.org/Top10/A02_2021-Cryptographic_Failures/)
- [CWE-798: Use of Hard-coded Credentials](https://cwe.mitre.org/data/definitions/798.html)

---

> **Suggested Hashnode tags:** `github`, `security`, `devops`, `githubactions`, `devsecops`, `git`, `webdev`, `programming`
>
> **Suggested cover image:** A terminal showing a `git log` output with a redacted credential, or a GitHub Actions workflow file.

*Found this useful? Share it with your team — especially the checklist before making any repo public. The five minutes it takes to run through it are worth considerably more than the incident response that follows if you skip it.*
