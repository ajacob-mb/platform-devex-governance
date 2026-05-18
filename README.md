# Labels Governance Platform (DevEx)

This repository provides a centralised platform to standardise and reconcile GitHub labels across all repositories accessible to a GitHub App installation.

---

## Purpose

- Standardise governed labels across all repositories
- Safely migrate legacy labels to the governance labels
- Prevent drift with optional scheduled reconciliation runs
- Guarantee every issue and open PR has at minimum one type and one priority label

---

## Config Files

| File | Purpose |
|---|---|
| `labels/config/labels.json` | Source of truth — all desired labels with name, color, and description |
| `labels/config/label-mapping.json` | Maps legacy label names to their governance label equivalents |
| `labels/config/staged-migration.json` | Defines `stage1` and `stage2` explicit repo lists for controlled rollout |
| `.github/workflows/labels-governance.yml` | Main workflow — introduce, migrate, and reconcile |
| `.github/workflows/reusable-create-branch.yml` | Reusable workflow — creates and links issue branches from `/create-branch` |
| `.github/workflows/reusable-flag-governance.yml` | Reusable workflow — enforces issue flag policy and safeguards |
| `.github/workflows/reusable-pr-lifecycle.yml` | Reusable workflow — syncs PR metadata with linked issues |
| `.github/workflows/reusable-rework.yml` | Reusable workflow — handles `/rework` issue commands |

---

## DevEx Workflow Split

This repository now separates the real-time DevEx automation into reusable workflows that are called from each target repository.

Central control repository:

```text
.github/workflows/
	labels-governance.yml
	reusable-create-branch.yml
	reusable-flag-governance.yml
	reusable-pr-lifecycle.yml
	reusable-rework.yml
```

Target repository:

```text
.github/workflows/
	devex.yml
```

The target repository owns the event triggers. The reusable workflows in this repository own the logic and run against the caller repository context.

Example target-repo caller:

```yaml
name: DevEx

on:
	issue_comment:
		types: [created]
	issues:
		types: [labeled, unlabeled]
	pull_request_target:
		types: [opened, edited, reopened, synchronize, closed]

permissions:
	contents: write
	issues: write
	pull-requests: write

jobs:
	create-branch:
		if: github.event_name == 'issue_comment'
		uses: org-or-owner/PlatformDevExGovernance/.github/workflows/reusable-create-branch.yml@main

	flag-governance:
		if: github.event_name == 'issues' || github.event_name == 'issue_comment'
		uses: org-or-owner/PlatformDevExGovernance/.github/workflows/reusable-flag-governance.yml@main

	pr-lifecycle:
		if: github.event_name == 'pull_request_target'
		uses: org-or-owner/PlatformDevExGovernance/.github/workflows/reusable-pr-lifecycle.yml@main

	rework:
		if: github.event_name == 'issue_comment'
		uses: org-or-owner/PlatformDevExGovernance/.github/workflows/reusable-rework.yml@main
```

Notes:

- The caller repository must grant the permissions needed by the reusable workflow jobs.
- `issue_comment` is intentionally shared by the create-branch, flag-governance, and rework flows; each reusable workflow filters its own commands and conditions.
- Replace `org-or-owner/PlatformDevExGovernance@main` with the correct repository and ref before templating target repositories.

---

## Label Groups

| Group | Labels |
|---|---|
| **type** | `type: bug`, `type: docs`, `type: feature`, `type: task` |
| **priority** | `priority: high`, `priority: medium`, `priority: low` |
| **flag** | `flag: rework`, `flag: blocked`, `flag: duplicate`, `flag: invalid`, `flag: wontfix` |

---

## Modes

### `introduce`
- Creates missing labels in target repositories
- Updates color and description of labels that have drifted
- Does not modify issues or PRs
- Does not delete labels

### `migrate`
- Everything `introduce` does
- Maps legacy labels on issues to their governance label equivalents
- Preserves existing labels — does not remove anything
- Applies to all issues and open PRs

### `reconcile`
- Everything `migrate` does
- Deletes any label in the repository not defined in `labels.json`
- Applies minimum policy: adds `priority: medium` to any work item with no priority label
- Applies minimum policy: adds `type: task` to any work item with no type label
- Minimum policy applies to all issues and open PRs; closed PRs are not modified
- Recommended interpretation: weekly reconciliation mode for drift prevention

---

## Stages

| Stage | Behaviour |
|---|---|
| `stage1` | Runs against the explicit repo list defined in `staged-migration.json` |
| `stage2` | Runs against the explicit repo list defined in `staged-migration.json` |
| `all` | Dynamically discovers and runs against every repository accessible to the GitHub App installation |

---

## Authentication

The workflow uses a GitHub App installation token for cross-repository operations.

Configure these repository (or organization) Actions secrets where this workflow runs:

- `GH_APP_ID`
- `GH_APP_PRIVATE_KEY`
- `GH_INSTALLATION_ID`

`GITHUB_TOKEN` is still present for workflow runtime, but label-governance API calls use the GitHub App token generated from the secrets above.

---

## Workflow Inputs

| Input | Required | Default (scheduled) | Description |
|---|---|---|---|
| `mode` | Yes | `reconcile` | Execution mode: `introduce`, `migrate`, or `reconcile` (drift prevention) |
| `stage` | Yes | `all` | Rollout scope: `stage1`, `stage2`, or `all` |
| `dry_run` | Yes | `false` | When `true`, logs all planned changes without writing anything |
| `reconcile_confirmation` | No | n/a | Reconcile mode only — leave blank for introduce/migrate. Type `RECONCILE` to confirm. |

---

## Recommended Manual Rollout Sequence

1. `introduce` + `stage1` + `dry_run=true` — preview label creation
2. `introduce` + `stage1` + `dry_run=false` — apply
3. `migrate` + `stage1` + `dry_run=true` — preview issue remapping
4. `migrate` + `stage1` + `dry_run=false` — apply
5. Repeat steps 1–4 for `stage2`
6. `reconcile` + `all` + `dry_run=true` — preview full reconciliation
7. `reconcile` + `all` + `dry_run=false` — apply full manual reconciliation across all app-accessible repositories

---

## Scheduled Reconciliation

Scheduled reconciliation is active using the cron line in `.github/workflows/labels-governance.yml`.

The workflow runs automatically every **Monday at 03:00 UTC** in `reconcile` mode across all app-accessible repositories with `dry_run=false`.

This is independent of manual runs. Running step 7 above does not enable schedule; schedule is already active whenever cron is enabled.

To pause the schedule, comment out the cron line in the workflow file and merge to the default branch.
To stop a run in progress, cancel it from the GitHub Actions UI.
To disable entirely, use the workflow disable toggle in the Actions tab.

---

## Audit Output

Every run produces a `labels-governance-audit.json` artifact attached to the workflow run. It includes:

- Run timestamp, mode, stage, and dry-run flag
- Per-repository counts: labels created, updated, deleted, work items scanned, mapped labels added, defaults applied
- Aggregate totals across all repositories

A summary is also written to the GitHub Actions step summary for quick review.

---

## Notes

- The workflow is idempotent — safe to run multiple times; only out-of-sync items are changed
- Dry-run mode produces the same log output and audit file as a live run, but makes no API writes
- Concurrency control prevents duplicate runs for the same stage and mode from overlapping
- The GitHub App token is scoped to the installation — the workflow only touches repositories the app is installed on
