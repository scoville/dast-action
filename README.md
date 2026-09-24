# dast-action

A reusable GitHub Actions workflow that runs an authenticated [HawkScan](https://docs.stackhawk.com/) DAST scan against a staging host, and reports the results.

## Features

- Installs the latest `hawk` CLI, verified against the manifest's SHA-256
- Signs in as a scan user (Devise form login, optional tenant `client_slug`)
- Sends a WAF bypass header so payloads reach the app. The key comes from a repo secret or from SSM
- Writes the findings as a table on the run page, with the offending paths behind `<details>`
- Uploads the raw alert JSON as the `dast-findings` artifact (kept 90 days)
- Posts a DCF-18 (Vulnerability Scans) evidence record to the Drata "HawkScan DAST" custom connection
- Pings Slack when the scan finds High-risk issues
- Fails the job when findings cross the `stackhawk.yml` threshold (hawk exit 42) or the scan itself fails

Each app keeps its own `stackhawk.yml` (seed paths, cookie name, excludes), because those are app-specific.

## Usage

Copy [`examples/caller.yml`](examples/caller.yml) to `.github/workflows/dast.yml` in the app repo:

```yaml
name: dast

on:
  schedule:
    - cron: '0 18 * * 0' # 03:00 JST Monday
  workflow_dispatch:
    inputs:
      host:
        description: 'Tenant host to scan (bare hostname, no scheme)'
        default: 'stg.noman-ai.jp'
      waf_bypass:
        description: 'Send the WAF bypass header so payloads reach the app. Off = scan as an outsider sees it.'
        type: boolean
        default: true

permissions:
  contents: read
  id-token: write # only for the SSM key source

concurrency:
  group: dast-${{ github.ref }}
  cancel-in-progress: false

jobs:
  scan:
    uses: scoville/dast-action/.github/workflows/dast.yml@v1
    secrets: inherit
    with:
      app_name: noman
      host: ${{ inputs.host || 'stg.noman-ai.jp' }}
      waf_bypass: ${{ github.event_name != 'workflow_dispatch' || inputs.waf_bypass }}
      waf_key_ssm_parameter: /noman-stg/dast/scan-key
      stackhawk_app_id_secret: STACKHAWK_APP_ID
      scan_username_secret: E2E_USERNAME
      scan_password_secret: E2E_PASSWORD
      client_slug_secret: E2E_CLIENT_SLUG
      slack_channel_id: ${{ vars.DAST_SLACK_CHANNEL_ID || 'C09LMU51P35' }}
```

The trigger, permissions and concurrency stay in the caller. A called workflow can only narrow the caller's permissions, so this workflow doesn't declare any.

## Secrets

Callers pass `secrets: inherit`. Secrets come in two kinds:

**Org-wide.** The workflow reads these by fixed name, and callers never mention them. Define them once as organization secrets. A repo secret with the same name also works and takes precedence.

| Secret | Required | Used for |
|--------|----------|----------|
| `HAWK_API_KEY` | Yes | StackHawk API key (scan + alert fetch) |
| `DRATA_CCT_API_KEY` | No | Drata custom-connection key. The Drata step is skipped if it's missing |
| `SLACK_BOT_TOKEN` | No | Slack bot token. The Slack step is skipped if it's missing |

**Repo-specific.** The caller names these through the `*_secret` inputs, and the workflow resolves them with `secrets[<name>]`:

| Input | Typical secret | Used for |
|-------|----------------|----------|
| `stackhawk_app_id_secret` | `STACKHAWK_APP_ID` | StackHawk application id |
| `scan_username_secret` | `E2E_USERNAME` | Scan user login |
| `scan_password_secret` | `E2E_PASSWORD` | Scan user password |
| `client_slug_secret` | `E2E_CLIENT_SLUG` | Tenant slug. The sign-in page is `/users/sign_in?client_slug=<slug>` |
| `waf_scan_key_secret` | `DAST_SCAN_KEY` | WAF bypass key (direct source) |
| `aws_role_secret` | `AWS_ROLE_TO_ASSUME_STG` | IAM role that reads the SSM key (SSM source) |

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `app_name` | Yes | - | Short app name. The Drata record id is `<app_name>-dast`, so keep it stable or a new record will be created |
| `host` | Yes | - | Host to scan, as a bare hostname |
| `stackhawk_app_id_secret` | Yes | - | See [Secrets](#secrets) |
| `scan_username_secret` | Yes | - | See [Secrets](#secrets) |
| `scan_password_secret` | Yes | - | See [Secrets](#secrets) |
| `client_slug_secret` | No | `''` | See [Secrets](#secrets) |
| `app_env` | No | `Staging` | StackHawk environment name |
| `config_file` | No | `stackhawk.yml` | HawkScan config in the caller repo |
| `login_path` | No | `/users/sign_in` | Sign-in path. The client slug is appended to it |
| `waf_bypass` | No | `true` | Send the WAF bypass header |
| `waf_scan_key_secret` | No | `''` | Repo secret holding the bypass key. Takes precedence over SSM |
| `waf_key_ssm_parameter` | No | `''` | SSM SecureString holding the bypass key |
| `aws_role_secret` | No | `AWS_ROLE_TO_ASSUME_STG` | See [Secrets](#secrets) |
| `aws_region` | No | `ap-northeast-1` | Region of the SSM parameter |
| `drata_connection_id` | No | `20` | Drata custom connection id |
| `drata_resource_id` | No | `1` | Drata custom connection resource id |
| `slack_channel_id` | No | `''` | Slack channel for High-finding alerts. Empty disables Slack |
| `artifact_retention_days` | No | `90` | Retention of the `dast-findings` artifact |
| `timeout_minutes` | No | `60` | Job timeout |

## Outputs

| Output | Description |
|--------|-------------|
| `high` | Number of High-risk findings. Empty if the alert fetch failed |
| `medium` | Number of Medium-risk findings |
| `low` | Number of Low-risk findings |

## WAF bypass key

The staging WAF lets a request skip its managed rule groups when it carries `x-dast-scan-key: <key>`. The key is a Terraform `random_password` published to SSM (`nomanbase_infra/modules/core_infrastructure/waf.tf`). The workflow picks its source in this order:

1. **Repo secret** (`waf_scan_key_secret`). No AWS involved, and the caller doesn't need `id-token: write`. If Terraform ever regenerates the key, this secret goes stale and scans quietly run with the WAF active (lots of 403s, few findings). Update it whenever the SSM value changes.
2. **SSM** (`waf_key_ssm_parameter`). The workflow assumes the role in `aws_role_secret` over OIDC and reads the current value, so it can't go stale.
3. **Neither.** The workflow logs a warning and scans with the WAF active.

When `waf_bypass` is off, the header is still sent, but with a placeholder value that can't match the rule.

## Adding a new repo

1. Create a StackHawk application for the app, and store its id as the `STACKHAWK_APP_ID` repo secret.
2. Make sure a scan user exists on staging, and store `E2E_USERNAME`, `E2E_PASSWORD` and `E2E_CLIENT_SLUG`.
3. Enable `enable_dast_scan_bypass` for the stage in the infra repo. Then either copy the SSM value into a `DAST_SCAN_KEY` secret, or give the repo an OIDC role (`AWS_ROLE_TO_ASSUME_STG`) that can read `/<app>-stg/dast/scan-key`.
4. Check that the org secrets `HAWK_API_KEY`, `DRATA_CCT_API_KEY` and `SLACK_BOT_TOKEN` are shared with the repo.
5. Copy [`examples/stackhawk.yml`](examples/stackhawk.yml) to the repo root. Set the cookie name and host, and take the seed paths from `config/routes.rb`.
6. Copy [`examples/caller.yml`](examples/caller.yml) to `.github/workflows/dast.yml`, then run it once from the Actions tab.

## Versioning

Callers pin `@v1`. Every release gets a `vX.Y.Z` tag, and the `v1` tag is moved to the latest non-breaking release:

```bash
git tag v1.0.1 && git tag -f v1 && git push origin v1.0.1 && git push -f origin v1
```

This repo is private. Settings → Actions → General → Access is set to "Accessible from repositories in the scoville organization" so other repos can call the workflow.
