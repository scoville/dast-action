# dast-action

A composite GitHub Action that runs an authenticated [HawkScan](https://docs.stackhawk.com/) DAST scan against a staging host, and reports the results.

## Features

- Installs the latest `hawk` CLI, verified against the manifest's SHA-256
- Signs in as a scan user (Devise form login, optional tenant `client_slug`)
- Sends a WAF bypass header so payloads reach the app. The key is passed in directly or read from SSM
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
        default: 'stg.example.jp'
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
    runs-on: ubuntu-latest
    timeout-minutes: 60
    steps:
      - name: DAST scan
        uses: scoville/dast-action@v1
        with:
          app-name: noman
          host: ${{ inputs.host || 'stg.example.jp' }}
          waf-bypass: ${{ github.event_name != 'workflow_dispatch' || inputs.waf_bypass }}
          waf-key-ssm-parameter: /noman-stg/dast/scan-key
          aws-role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME_STG }}
          stackhawk-api-key: ${{ secrets.HAWK_API_KEY }}
          stackhawk-app-id: ${{ secrets.STACKHAWK_APP_ID }}
          scan-username: ${{ secrets.DAST_USERNAME }}
          scan-password: ${{ secrets.DAST_PASSWORD }}
          client-slug: ${{ secrets.DAST_CLIENT_SLUG }}
          drata-api-key: ${{ secrets.DRATA_CCT_API_KEY }}
          slack-bot-token: ${{ secrets.SLACK_BOT_TOKEN }}
          slack-channel-id: ${{ vars.DAST_SLACK_CHANNEL_ID }}
```

The action checks out the caller repo itself, so it can read `stackhawk.yml`; the caller doesn't need an `actions/checkout` step. The trigger, permissions, concurrency and job timeout stay in the caller.

## Inputs

Secrets are passed as ordinary inputs. They stay explicit, since the action only receives what you hand it, and GitHub masks them in logs. The suggested repo secret names are only a convention, so map whatever your repo already has. For example, the nomanbase web repos pass their existing `E2E_*` user.

| Input | Required | Default | Suggested repo secret | Description |
|-------|----------|---------|-----------------------|-------------|
| `app-name` | Yes | - | | Short app name. The Drata record id is `<app-name>-dast`, so keep it stable or a new record will be created |
| `host` | Yes | - | | Host to scan, as a bare hostname |
| `stackhawk-api-key` | Yes | - | `HAWK_API_KEY` (shared value, provided by us) | StackHawk API key (scan + alert fetch) |
| `stackhawk-app-id` | Yes | - | `STACKHAWK_APP_ID` | StackHawk application id |
| `scan-username` | Yes | - | `DAST_USERNAME` | Scan user login |
| `scan-password` | Yes | - | `DAST_PASSWORD` | Scan user password |
| `client-slug` | No | `''` | `DAST_CLIENT_SLUG` | Tenant slug. The sign-in page becomes `<login-path>?client_slug=<slug>` |
| `app-env` | No | `Staging` | | StackHawk environment name |
| `config-file` | No | `stackhawk.yml` | | HawkScan config in the caller repo |
| `login-path` | No | `/users/sign_in` | | Sign-in path |
| `waf-bypass` | No | `true` | | Send the WAF bypass header |
| `waf-scan-key` | No | `''` | `DAST_SCAN_KEY` | WAF bypass key. Takes precedence over SSM |
| `waf-key-ssm-parameter` | No | `''` | | SSM SecureString holding the bypass key |
| `aws-role-to-assume` | No | `''` | `AWS_ROLE_TO_ASSUME_STG` | IAM role that reads the SSM key over OIDC |
| `aws-region` | No | `ap-northeast-1` | | Region of the SSM parameter |
| `drata-api-key` | No | `''` | `DRATA_CCT_API_KEY` (shared value, provided by us) | Drata custom-connection key. The Drata step is skipped when it's empty |
| `drata-connection-id` | No | `20` | | Drata custom connection id |
| `drata-resource-id` | No | `1` | | Drata custom connection resource id |
| `slack-bot-token` | No | `''` | `SLACK_BOT_TOKEN` (shared bot, or your own) | Slack bot token. Slack is skipped when it's empty |
| `slack-channel-id` | No | `''` | | Slack channel for High-finding alerts. Slack is skipped when it's empty |
| `artifact-retention-days` | No | `90` | | Retention of the `dast-findings` artifact |

GitHub doesn't enforce `required` for actions, so the first step fails with a clear error when a required input is empty. That usually means the secret is missing from the repo.

## Outputs

| Output | Description |
|--------|-------------|
| `high` | Number of High-risk findings. Empty if the alert fetch failed |
| `medium` | Number of Medium-risk findings |
| `low` | Number of Low-risk findings |

## WAF bypass key

The staging WAF lets a request skip its managed rule groups when it carries `x-dast-scan-key: <key>`. The key is a Terraform `random_password` published to SSM (`nomanbase_infra/modules/core_infrastructure/waf.tf`). The action picks its source in this order:

1. **`waf-scan-key`**. No AWS involved, and the caller doesn't need `id-token: write`. If Terraform ever regenerates the key, the secret goes stale and scans quietly run with the WAF active (lots of 403s, few findings). Update it whenever the SSM value changes.
2. **SSM** (`waf-key-ssm-parameter` + `aws-role-to-assume`). The action assumes the role over OIDC and reads the current value, so it can't go stale.
3. **Neither.** The action logs a warning and scans with the WAF active.

When `waf-bypass` is off, the header is still sent, but with a placeholder value that can't match the rule.

## Adding a new repo

1. Create a StackHawk application for the app, and store its id as the `STACKHAWK_APP_ID` repo secret.
2. Make sure a scan user exists on staging, and store `DAST_USERNAME`, `DAST_PASSWORD` and `DAST_CLIENT_SLUG`.
3. Enable `enable_dast_scan_bypass` for the stage in the infra repo. Then either copy the SSM value into a `DAST_SCAN_KEY` secret, or give the repo an OIDC role (`AWS_ROLE_TO_ASSUME_STG`) that can read `/<app>-stg/dast/scan-key`.
4. Add the shared `HAWK_API_KEY` and `DRATA_CCT_API_KEY` values we provide (plus `SLACK_BOT_TOKEN` for alerts) as repo secrets.
5. Copy [`examples/stackhawk.yml`](examples/stackhawk.yml) to the repo root. Set the cookie name and host, and take the seed paths from `config/routes.rb`.
6. Copy [`examples/caller.yml`](examples/caller.yml) to `.github/workflows/dast.yml` and fill in the inputs, then run it once from the Actions tab.

## Versioning

Callers pin `@v1`. Every release gets a `vX.Y.Z` tag, and the `v1` tag is moved to the latest non-breaking release:

```bash
git tag v1.0.1 && git tag -f v1 && git push origin v1.0.1 && git push -f origin v1
```

This repo is public, so any repository can use the action without an Actions access setting.

## Contributing

This action is maintained by Scoville for its own apps. We don't accept pull requests from outside the scoville organization, and workflows on outside pull requests only run after a maintainer approves them.
