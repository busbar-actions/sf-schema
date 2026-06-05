> [!WARNING]
> **`busbar-actions` is under heavy active development — expect breaking changes.**
> These repositories are public, but **not ready for use yet** — please don't depend on them.
> A pilot is starting soon: **[star and watch the busbar-actions organization](https://github.com/busbar-actions)** for the launch of Discussions and the pilot announcement.

# busbar-actions/sf-schema

Pull a Salesforce org's schema and write it into your repo as **per-SObject JSON describe files**, so schema drift surfaces as reviewable git diffs.

Each SObject lands as `<output-path>/<SObjectName>.json` containing the full REST Describe response (pretty-printed, stably ordered), giving you a diff-friendly snapshot of every object's fields, relationships, picklists, etc.

## What it does

Installs and runs the `sf-schema-dump` binary, which:

1. Authenticates to Salesforce via `busbar-auth` (`session_from_env`). **By default it self-mints**: when no `sf-access-token` is supplied, the binary exchanges the runner's GitHub OIDC id-token in-process for a short-lived session against the Busbar-equipped org at `target-instance`, holds it only in zeroizing memory, and revokes + zeroizes it (`session.dispose()`) at exit. If you instead pass `sf-access-token` + `sf-instance-url`, it uses that handed-off token directly and skips OIDC.
2. Resolves the candidate SObject list — either the explicit `objects` list, or the org's global SObject list (`GET /services/data/vXX/sobjects/`).
3. Filters candidates by capability flags (`filters`) and/or an API-name glob (`name-match`).
4. Calls REST Describe on each selected SObject and writes the pretty JSON to `<output-path>/<Name>.json`.
5. When `commit` is `true`, commits the refreshed schema back to the current branch.
6. Emits GitHub outputs, a job summary table, and warning annotations for any SObjects that failed describe.

The implementation is typesynth-free — pure `reqwest` against `/services/data/vXX/sobjects/<Name>/describe` — so the binary stays small and builds without the schema-modeling crate.

## Usage (OIDC self-mint — default, recommended)

No Salesforce token in GitHub: the binary mints its own short-lived token from the runner's OIDC id-token. **Grant `permissions: id-token: write`** and point `target-instance` at your Busbar-equipped org.

```yaml
permissions:
  contents: write   # required only when commit: true
  id-token: write   # required for the OIDC token exchange

steps:
  - uses: actions/checkout@v4

  - uses: busbar-actions/sf-schema@v1
    with:
      target-instance: https://acme.my.salesforce.com
      filters: custom
      name-match: 'Account*'
```

The org must trust this repo (`sf busbar trust request approve`); a first run from a new repo/workflow stops with a pending-trust error until approved.

### Local-dev / advanced override (handed-off token)

If you already have a session token (e.g. local testing), set both `sf-access-token` and `sf-instance-url`; the binary uses them directly and skips OIDC. Do not use this in CI — prefer the OIDC path so no SF token is ever passed through the workflow.

```yaml
- uses: busbar-actions/sf-schema@v1
  with:
    sf-access-token: ${{ secrets.SF_ACCESS_TOKEN }}
    sf-instance-url: ${{ secrets.SF_INSTANCE_URL }}
    filters: custom
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `target-instance` | yes* | `` | Busbar-equipped org instance URL for the OIDC self-mint (e.g. `https://acme.my.salesforce.com`). *Required unless you supply the `sf-access-token`/`sf-instance-url` override. Requires `permissions: id-token: write`. |
| `eca-client-id` | no | `` | Override the External Client App consumer key for the OIDC exchange. PBO-pinned default; set only on a rotation. |
| `token-handler` | no | `` | Override the Apex token-exchange handler dev name. Defaults to `BBGitHubTokenExchangeHandler`. |
| `oidc-audience` | no | `` | Override the audience requested in the GitHub OIDC token. Defaults to `target-instance`. |
| `sf-access-token` | no | `` | OPTIONAL local-dev/advanced override: a handed-off Salesforce session access token. When set (with `sf-instance-url`), the binary uses it directly and skips OIDC. |
| `sf-instance-url` | no | `` | OPTIONAL override paired with `sf-access-token`, e.g. `https://acme.my.salesforce.com`. |
| `output-path` | no | `.busbar/Platforms/Salesforce/Schema` | Where the per-SObject `.json` describe files are written, relative to the repo root. |
| `filters` | no | `` | Comma-separated capability flags; all must match. Valid values: `createable`, `updateable`, `deletable`, `queryable`, `retrieveable`, `searchable`, `custom`, `standard`. |
| `name-match` | no | `` | Glob over SObject API names (e.g. `Account*`, `*__c`). Applied in addition to `filters`. |
| `objects` | no | `` | Explicit comma-separated SObject list. Overrides `filters` / `name-match` when set. |
| `version` | no | `latest` | `sf-schema-dump` release tag to download (e.g. `v0.4.2`). `latest` resolves the most recent release. |
| `binary-repo` | no | `busbar-actions/actions-dist` | Repo that publishes the `sf-schema-dump` binary releases. |
| `commit` | no | `true` | Commit any changes back to the current branch after the pull. |
| `commit-message` | no | `chore(schema): refresh Salesforce schema` | Commit message used when changes are committed. |
| `git-user-name` | no | `busbar-bot` | `git user.name` for the commit. |
| `git-user-email` | no | `bot@busbar.agency` | `git user.email` for the commit. |

## Outputs

| Output | Description |
|---|---|
| `changed` | `"true"` if the schema export produced changes that were committed (or would have been). |
| `object-count` | Number of describe JSON files written. |
| `selected-count` | Number of SObjects selected after filtering. |
| `failed-count` | Number of SObjects that failed describe. |
| `output-path` | Path where the schema was written (echoes the input). |

## Authentication and permissions

**Default (recommended): GitHub OIDC self-mint — zero stored SF credentials.** A Salesforce access token is never handed to a script, written to `GITHUB_ENV`, passed as an input/output, or persisted. The binary mints its OWN short-lived token in-process from the runner's GitHub OIDC id-token (exchanged at the Busbar-equipped org's Apex token handler against `target-instance`), holds it only in zeroizing memory (`busbar-auth` `CredentialContext`), uses it, and **revokes + zeroizes it (`session.dispose()`) at exit** because the token is OIDC-minted-and-owned.

To enable it, the caller job MUST grant `id-token: write`:

```yaml
permissions:
  contents: write   # required only when `commit: true`
  id-token: write   # required for the OIDC token exchange (the default auth path)
```

The org must trust this repo. On the first run from a new repo/workflow the exchange returns a pending-trust error; approve it (`sf busbar trust request approve`, which creates + activates the trust rule) and re-run.

**Optional override (local-dev / advanced): handed-off token.** Set `sf-access-token` + `sf-instance-url` and the binary uses that session directly, skipping OIDC. Because such a token is handed off and not minted by this action, the binary does **not** revoke it — it only zeroizes its in-memory copy on drop. The token's lifecycle is the caller's responsibility. How you mint it is up to you (e.g. `sf org display --json` after `sf org login web`, stored in a secret). Do not use this path in CI when OIDC is available.

## Example: `workflow_dispatch` consumer

Save as `.github/workflows/sf-schema-refresh.yml` in the consuming repo:

```yaml
name: Refresh SF Schema

on:
  workflow_dispatch:
    inputs:
      filters:
        description: 'SObject filters (comma-separated). Leave blank for all.'
        required: false
        default: 'custom'
      name-match:
        description: 'Glob over SObject names (e.g. Account*)'
        required: false
        default: ''

permissions:
  contents: write
  id-token: write   # OIDC self-mint

jobs:
  refresh:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: busbar-actions/sf-schema@v1
        with:
          target-instance: https://acme.my.salesforce.com
          filters: ${{ inputs.filters }}
          name-match: ${{ inputs.name-match }}
```

## Example: nightly refresh

```yaml
name: Nightly SF Schema Refresh

on:
  schedule:
    - cron: '0 6 * * *'   # 06:00 UTC daily

permissions:
  contents: write
  id-token: write   # OIDC self-mint

jobs:
  refresh:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: busbar-actions/sf-schema@v1
        with:
          target-instance: https://acme.my.salesforce.com
          commit-message: 'chore(schema): nightly refresh'
```

## Observability

The binary owns its UX through the `github-actions-ux` crate:

- Writes all GitHub outputs (`changed`, `object-count`, `selected-count`, `failed-count`, `output-path`) and a job-summary table via the `Reporter`.
- Emits warning annotations (capped at 10) for SObjects that failed describe.
- On a fatal error it sets the step failed and exits non-zero.

Known gaps (tracked for a follow-up): status lines and some warnings (auth instance URL, "no SObjects matched", describe-failure counts) are still plain `eprintln!` rather than `Reporter` annotations, the crate does not use `tracing`, and `main` uses `fail()` rather than `run_outcome` + `RecordingReporter`, so a hard failure is annotated but not guaranteed into the Job Summary.

## Binary distribution

This composite action downloads an `sf-schema-dump` release binary from `binary-repo` (default `busbar-actions/actions-dist`) via the shared `busbar-actions/setup` install action, then runs it with all inputs passed through as `INPUT_*` / `SF_*` env.
