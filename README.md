# busbar-actions/sf-schema

Pull a Salesforce org's schema and write it into your repo as **per-SObject JSON describe files**, so schema drift surfaces as reviewable git diffs.

Each SObject lands as `<output-path>/<SObjectName>.json` containing the full REST Describe response (pretty-printed, stably ordered), giving you a diff-friendly snapshot of every object's fields, relationships, picklists, etc.

## What it does

Wraps the `busbar-sf schema dump` command, which:

1. Authenticates to Salesforce via session token (`SF_ACCESS_TOKEN` + `SF_INSTANCE_URL`).
2. Fetches the org's SObject list (optionally filtered by capability / name glob / explicit object list).
3. Calls REST Describe on each selected SObject and writes the JSON to `<output-path>/<Name>.json`.
4. (Optionally) commits the result back to the current branch.

The implementation is typesynth-free — pure `reqwest` against `/services/data/vXX/sobjects/<Name>/describe` — so the binary stays small and builds without the schema-modeling crate.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `sf-access-token` | yes | — | SF session token. Provide via a secret. |
| `sf-instance-url` | yes | — | e.g. `https://acme.my.salesforce.com` |
| `output-path` | no | `.busbar/Platforms/Salesforce/Schema` | Where the per-SObject `.kant` files are written. |
| `filters` | no | `` | Comma-separated flags: `custom`, `standard`, `populated`, `extractable`, `has-master-detail`, `has-record-types`, `has-formula-fields`, `has-encrypted-fields`, `createable`, `updateable`, `deletable`, `queryable`, `retrieveable`, `searchable`, `is-lookup-target` |
| `name-match` | no | `` | Glob over SObject API names (e.g. `Account*`, `*__c`) |
| `objects` | no | `` | Explicit comma-separated SObject list. Overrides `filters` / `name-match` when set. |
| `min-field-count` | no | `` | Drop SObjects with fewer than N fields. |
| `max-field-count` | no | `` | Drop SObjects with more than N fields. |
| `include-counts` | no | `false` | Include record counts (one extra API call per SObject). |
| `version` | no | `latest` | `busbar-sf` release tag to download. |
| `binary-repo` | no | `busbar-actions/actions-dist` | Repo that publishes `busbar-sf` binary releases. |
| `commit` | no | `true` | Commit any changes back to the current branch. |
| `commit-message` | no | `chore(schema): refresh Salesforce schema` | |
| `git-user-name` | no | `busbar-bot` | |
| `git-user-email` | no | `bot@busbar.agency` | |

## Outputs

| Output | Description |
|---|---|
| `changed` | `"true"` if the export produced file changes. |
| `object-count` | Number of SObjects written. |
| `output-path` | Echo of the input. |

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
        default: 'custom,has-record-types'
      name-match:
        description: 'Glob over SObject names (e.g. Account*)'
        required: false
        default: ''
      include-counts:
        description: 'Include record counts'
        required: false
        type: boolean
        default: false

permissions:
  contents: write

jobs:
  refresh:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: busbar-actions/sf-schema@v1
        with:
          sf-access-token: ${{ secrets.SF_ACCESS_TOKEN }}
          sf-instance-url: ${{ secrets.SF_INSTANCE_URL }}
          filters: ${{ inputs.filters }}
          name-match: ${{ inputs.name-match }}
          include-counts: ${{ inputs.include-counts }}
```

## Example: nightly refresh

```yaml
name: Nightly SF Schema Refresh

on:
  schedule:
    - cron: '0 6 * * *'   # 06:00 UTC daily

permissions:
  contents: write

jobs:
  refresh:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: busbar-actions/sf-schema@v1
        with:
          sf-access-token: ${{ secrets.SF_ACCESS_TOKEN }}
          sf-instance-url: ${{ secrets.SF_INSTANCE_URL }}
          commit-message: 'chore(schema): nightly refresh'
```

## Authentication

For v1 the action takes a session token + instance URL via secrets:

- `SF_ACCESS_TOKEN` — a live Salesforce session access token.
- `SF_INSTANCE_URL` — e.g. `https://acme.my.salesforce.com`.

How you mint that token is up to you. Common patterns:

- `sf org display --target-org <alias> --json` after `sf org login web` (interactive setup, store result in secrets).
- `sf org login jwt ...` against a Connected App with a server certificate (headless; bake into a preceding workflow step that exports `SF_ACCESS_TOKEN`/`SF_INSTANCE_URL` to `$GITHUB_ENV`).

A future release will accept the busbar OIDC trust handshake from a Connected-App-side TokenExchangeHandler so no SF secret needs to live in GitHub at all.

## Binary distribution (current status)

This composite action downloads a `busbar-sf` release binary from `binary-repo` (default: `busbar-actions/actions-dist`).

**Dependency to land**: `busbar-actions/actions-dist` does not yet exist and isn't being published to. To make this action work end-to-end, we need:

1. A `busbar-actions/actions-dist` repo (empty source, used purely as a release host).
2. A CI workflow in `busbar-extensions` that on tag push (or workflow_dispatch) builds `busbar-sf` (`cargo build --release -p busbar-sf --bin busbar-sf`) for the supported targets — `x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu`, `x86_64-apple-darwin`, `aarch64-apple-darwin`, `x86_64-pc-windows-msvc` — and creates a release in `busbar-actions/actions-dist` with those binaries attached using the asset names this action expects (see `action.yml` → "Determine asset name"). Cross-repo publishing requires a PAT or GitHub App installed on both repos.

Once that's in place, tag this action `v1` and consumers can pin it.
