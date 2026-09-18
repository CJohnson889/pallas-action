# Pallas accessibility check (GitHub Action)

Scans the pages a pull request deployed with the same rule set and severity mapping Pallas uses in the product, and fails the check when a **new** finding at or above a severity appears. Roadmap Block 5, item 24 (F13).

## What it does

1. Opens each URL you list in a real browser (Playwright Chromium, installed by the action).
2. Runs axe-core 4.11.3 under the standards profile you choose, with the same tag selection and the same explicit-rule additions the product uses (`src/engine/wcag/severityMap.ts`), and maps the results with the product's mapper (`src/engine/website/resultMapper.ts`), so a finding here is the finding Pallas would record.
3. Compares the findings with the repository's baseline (`.pallas/baseline.json` by default). A finding is the same finding across runs when its page, rule and element selector match, the identity the product's regression alerts use.
4. Writes a JSON report and a step summary, and fails when any **new** finding is at or above `fail-on` (default `critical`).

Findings already in the baseline never fail the check, however severe: the check is a ratchet against getting worse, not a gate on the backlog. Update the baseline on your default branch (`update-baseline: true`), never from a pull request, so a pull request cannot baseline its own regressions.

The URLs must be public, named, standard-port http(s) pages: the action applies the same SSRF guard the product's scanner does, so a preview deployment on a public hostname works and an internal address does not.

## Usage

```yaml
name: Accessibility
on:
  pull_request:
  push:
    branches: [main]

jobs:
  pallas:
    runs-on: ubuntu-latest
    permissions:
      contents: write   # only if you commit the baseline from main
    steps:
      - uses: actions/checkout@v5

      # Deploy or start your preview first, then point the check at it.
      - name: Pallas accessibility check
        id: pallas
        uses: CJohnson889/pallas-action@v1
        with:
          urls: |
            https://preview-${{ github.event.number }}.example.com/
            https://preview-${{ github.event.number }}.example.com/pricing
          profile: wcag_2_2_aa
          fail-on: critical
          baseline: .pallas/baseline.json
          update-baseline: ${{ github.ref == 'refs/heads/main' }}
          # Optional: record each run in your Pallas organization.
          pallas-token: ${{ secrets.PALLAS_TOKEN }}

      - name: Upload the report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: pallas-report
          path: ${{ steps.pallas.outputs.report }}

      - name: Commit the refreshed baseline (main only)
        if: github.ref == 'refs/heads/main'
        run: |
          git config user.name "pallas-bot"
          git config user.email "pallas-bot@users.noreply.github.com"
          git add .pallas/baseline.json
          git diff --cached --quiet || git commit -m "chore: refresh Pallas baseline" && git push
```

## Inputs

| Input | Default | Meaning |
|---|---|---|
| `urls` | required | One URL per line. |
| `profile` | `wcag_2_2_aa` | `wcag_2_2_a`, `wcag_2_2_aa`, `wcag_2_2_aaa`, `section_508`, `en_301_549`. |
| `fail-on` | `critical` | Fail on new findings at or above: `critical`, `major`, `minor`, `advisory`. |
| `baseline` | `.pallas/baseline.json` | Findings already known. Missing file means everything is new. |
| `update-baseline` | `false` | Write the current findings to the baseline path. |
| `report` | `pallas-report.json` | Where the JSON report goes. |
| `pallas-token` | none | An organization API token (Pallas Admin Settings, Growth and above). When set, the run is recorded in that organization as a scan. Store it as a repository secret and pass `${{ secrets.PALLAS_TOKEN }}`. |
| `pallas-api-url` | `https://pallas-app-api.lonia.ai` | Where the upload goes. Leave it unless Pallas support gives you another address. |

## Outputs

`critical-count` (all critical findings this run), `new-count` (not in the baseline), `failing-count` (new and at or above `fail-on`), `report` (absolute path), `pallas-scan-id` (the recorded scan when `pallas-token` is set and the upload was accepted; empty otherwise).

## What it does not do

Without `pallas-token` it sends nothing to Pallas and needs no account: the check runs entirely in your workflow. With a token, each run is uploaded to `POST /ci/upload` and recorded in your organization as a completed scan (source `github_action`, no initiating user) with its findings under your plan's finding cap, so it appears in the compliance score, the readiness page and the audit trail. The upload is idempotent per run attempt and is best effort: a refused or failed upload is a warning in the run log and never changes the check's result. Ten active tokens per organization, thirty uploads per hour per token.

## Source and releases

This repository holds the built action. The source is the `action/` directory of the Pallas application repository, where it is bundled with the product's own rule selection, severity mapping and result mapper; the bundle is exported here by `npm run action:export` and never edited by hand. `v1` is the moving major tag; `v1.0.0` is the first release.
