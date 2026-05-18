# Label Taxonomy Platform (DevEx)

This repository provides a centralised platform to standardise and enforce GitHub labels across all repositories accessible to a GitHub App installation.

---

## Purpose

- Standardise label taxonomy across all repositories
- Safely migrate legacy labels to the taxonomy
- Enforce consistency with optional scheduled runs
- Guarantee every issue and open PR has at minimum one type and one priority label

---

## Config Files

| File | Purpose |
|---|---|
| `labels/config/labels.json` | Source of truth — all desired labels with name, color, and description |
| `labels/config/label-mapping.json` | Maps legacy label names to their taxonomy equivalents |
| `labels/config/staged-migration.json` | Defines `stage1` and `stage2` explicit repo lists for controlled rollout |
| `.github/workflows/labels-governance.yml` | Main workflow — introduce, migrate, enforce |

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
- Maps legacy labels on issues to their taxonomy equivalents
- Preserves existing labels — does not remove anything
- Applies to all issues and open PRs

### `enforce`
- Everything `migrate` does
- Deletes any label in the repository not defined in `labels.json`
- Applies minimum policy: adds `priority: medium` to any work item with no priority label
- Applies minimum policy: adds `type: task` to any work item with no type label
- Minimum policy applies to all issues and open PRs; closed PRs are not modified

---

## Stages

| Stage | Behaviour |
|---|---|
| `stage1` | Runs against the explicit repo list defined in `staged-migration.json` |
| `stage2` | Runs against the explicit repo list defined in `staged-migration.json` |
| `all` | Dynamically discovers and runs against every repository accessible to the GitHub App installation |

---

## Workflow Inputs

| Input | Required | Default (scheduled) | Description |
|---|---|---|---|
| `mode` | Yes | `enforce` | Execution mode: `introduce`, `migrate`, or `enforce` |
| `stage` | Yes | `all` | Rollout scope: `stage1`, `stage2`, or `all` |
| `dry_run` | Yes | `false` | When `true`, logs all planned changes without writing anything |
| `enforce_confirmation` | No | n/a | Enforce mode only — leave blank for introduce/migrate. Type `ENFORCE` to confirm. |

---

## Recommended Manual Rollout Sequence

1. `introduce` + `stage1` + `dry_run=true` — preview label creation
2. `introduce` + `stage1` + `dry_run=false` — apply
3. `migrate` + `stage1` + `dry_run=true` — preview issue remapping
4. `migrate` + `stage1` + `dry_run=false` — apply
5. Repeat steps 1–4 for `stage2`
6. `enforce` + `all` + `dry_run=true` — preview full enforcement
7. `enforce` + `all` + `dry_run=false` — apply full manual enforcement across all app-accessible repositories

---

## Scheduled Enforcement

Scheduled enforcement is active using the cron line in `.github/workflows/labels-governance.yml`.

The workflow runs automatically every **Monday at 03:00 UTC** in `enforce` mode across all app-accessible repositories with `dry_run=false`.

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
