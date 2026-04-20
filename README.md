# SecureCheck

Reusable GitHub Actions workflow - runs a security scanner stack on every push and PR, posts a single summary to Discord.

---

## Features

- **One workflow, many repos** - thin caller in each consumer repo, scanner stack lives here; edit once, updates everywhere
- **Gitleaks** - hardcoded secrets in the working tree and history, runs on every push and PR
- **Semgrep** - pattern-based SAST using the `auto` ruleset
- **Trivy** - dependency CVEs and IaC / config misconfigurations
- **Claude review (optional)** - Sonnet 4.6 reviews the PR diff for logic bugs scanners miss; gated on `ANTHROPIC_API_KEY`, skipped silently when not set
- **Single Discord embed per run** - severity-colored, deduplicated, with direct links to the workflow run and pull request
- **Heartbeat on pull requests** - an embed is posted even on clean PR runs so reviewers can see the bot ran
- **Silent on clean pushes** - no noise for green `main` commits
- **Raw reports archived** - every run uploads the raw JSON for each scanner as an artifact (14-day retention)

---

## Install

Drop a thin caller workflow into the consumer repo at `.github/workflows/security.yml`:

```yaml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  scan:
    uses: w1ck3ds0d4/SecureCheck/.github/workflows/scan.yml@main
    secrets: inherit
```

Set the required secret on the consumer repo:

- `DISCORD_WEBHOOK_URL` - incoming webhook for the channel the bot should post to. Create under **Discord channel -> Integrations -> Webhooks -> New Webhook -> Copy Webhook URL**.

Optional secret for the Claude review step:

- `ANTHROPIC_API_KEY` - when present, the Claude review step runs on PRs. When absent, the step is skipped with no error.

The reusable workflow accepts two optional inputs:

```yaml
    uses: w1ck3ds0d4/SecureCheck/.github/workflows/scan.yml@main
    secrets: inherit
    with:
      node_version: '22'
      python_version: '3.12'
```

---

## Usage

### Push to `main`

Scans run, findings are tallied, and the Discord notifier stays silent unless at least one scanner reports something. If anything fires, a single embed is posted with per-scanner counts and a link to the workflow run.

### Pull request

Same scan pipeline runs, but the embed is always posted - green when clean, colored when not - so the reviewer has a clear signal before approving. Claude review (if enabled) runs only on pull requests; pushes skip it to save API calls.

### Severity colouring

| State | Colour |
|---|---|
| All scanners clean | Green |
| Findings, no gitleaks hit | Yellow |
| Many findings (>10 total) | Orange |
| Gitleaks hit | Red |

### Claude review findings

When the optional Claude step runs and reports critical or high-severity issues, the first five are inlined directly in the Discord embed with file, line, and a one-sentence explanation of the exploit path. Lower-severity Claude findings only appear in the run artifact.

### Raw reports

Every run uploads a `security-reports` artifact containing the raw JSON from each scanner (gitleaks, semgrep, trivy, claude) with 14-day retention. Useful when the embed count is non-zero and you want the full picture without re-running the scan locally.

---

## Project Structure

```
SecureCheck/
  .github/
    workflows/
      scan.yml                    Reusable workflow; checkout, scanners, notify, upload
  scripts/
    notify.mjs                    Builds the Discord embed from scanner counts + posts it
    claude-review.mjs             Sends the PR diff to Claude and emits structured findings
  package.json                    @anthropic-ai/sdk dependency for the optional Claude step
  LICENSE                         AGPL v3
  COMMERCIAL.md                   Commercial license terms
```

---

## License

This project is dual-licensed:

- **[AGPL v3](LICENSE)** - free for open-source use. Derivatives and SaaS deployments must release their source under AGPL.
- **[Commercial license](COMMERCIAL.md)** - for proprietary / closed-source use or hosted services that do not want to comply with AGPL source-disclosure requirements. Contact for terms.
