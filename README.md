# security-workflows

Reusable GitHub Actions workflow that runs a standard security-scan stack on every push and pull request, and posts a single summary to a Discord channel.

## What it runs

| Scanner | Purpose |
|---|---|
| [gitleaks](https://github.com/gitleaks/gitleaks) | Hardcoded secrets in the working tree and history |
| [Semgrep](https://semgrep.dev) | Pattern-based SAST using the `auto` config |
| [Trivy](https://github.com/aquasecurity/trivy) | Dependency CVEs + IaC/config misconfigurations |
| Claude Sonnet 4.6 (optional) | LLM review of the PR diff for logic bugs that pattern scanners miss |

Scanners run in sequence inside a single job. Their output is aggregated and summarized in one Discord embed per run.

## How findings are reported

- **Push to `main`**: silent on clean, embed posted on any finding.
- **Pull request**: an embed is always posted (clean runs included) so reviewers can see the bot ran.
- **Severity colour**: green (clean) · yellow (few findings) · orange (many) · red (gitleaks hit).
- **High/critical Claude findings** are inlined in the embed with file + line + one-sentence why.
- Raw JSON reports are uploaded as a workflow artifact (`security-reports`, 14-day retention).

## Use it in a repo

Add `.github/workflows/security.yml`:

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
    uses: w1ck3ds0d4/security-workflows/.github/workflows/scan.yml@main
    secrets: inherit
```

Then set these repo secrets:

- `DISCORD_WEBHOOK_URL` — **required** for notifications. Create an incoming webhook in your Discord channel (Integrations → Webhooks → New Webhook → Copy Webhook URL).
- `ANTHROPIC_API_KEY` — *optional*. If present, the Claude review step runs on PRs. If absent, the step is skipped silently.

## Tweak defaults

The reusable workflow takes two inputs, both optional:

```yaml
jobs:
  scan:
    uses: w1ck3ds0d4/security-workflows/.github/workflows/scan.yml@main
    secrets: inherit
    with:
      node_version: '22'
      python_version: '3.12'
```

## License

AGPL v3 with a commercial-licence option at `daniel.svs@outlook.com`, matching the rest of my repos.
