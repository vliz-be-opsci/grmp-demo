# GRMP Demonstrators

A repository containing demonstration configuration files and the orchestration workflow for the GRMP test system. The workflow runs the GRMP orchestrator, collects JUnit XML reports, publishes them to a dashboard, and optionally creates GitHub issues for failing test suites.

The orchestrator can be found [here](https://github.com/vliz-be-opsci/grmp-framework). Available test implementations can be found [here](https://github.com/vliz-be-opsci/grmp-test-implementations).

---

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── orchestrate-config-tests.yml   ← main orchestration workflow
├── dashboard/
│   └── index.html                         ← dashboard (pushed to report branch on each run)
├── input-echo-demo/                       ← simple demonstrator using input-echo-test
├── resource-analyses/                     ← resource availability, certificate, CORS and content negotiation checks
└── shacl-example/                         ← SHACL validation demonstration
```

---

## Available Demonstrators

### `input-echo-demo`
A minimal demonstration of the GRMP framework using the input-echo-test. It does not test any external resource — it simply verifies that configuration parameters are passed correctly through the orchestrator to the test container. Useful for validating a new setup.

### `resource-analyses`
A practical collection of tests for monitoring external resources. Uses the resource-availability, check-certificate, cors-compliance and content-negotiation test implementations to verify that configured URLs are reachable, have valid certificates, correctly advertise CORS headers, and properly respond to content negotiation requests.

### `shacl-example`
An early demonstration of the shacl-validation test, verifying that RDF graphs harvested from configured data URLs conform to a given SHACL shapes graph.

---

## Workflow

The workflow is triggered manually via `workflow_dispatch`. It runs the orchestrator against a selected configuration directory, pushes the resulting report to a dedicated report branch, and performs a series of post-processing steps.

### Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `report_branch` | Yes | `grmp-reports` | Branch to push reports and dashboard files to |
| `config_directory` | Yes | `input-echo-demo` | Top-level folder containing the YAML config files to run |
| `dashboard_label` | No | (repository name) | Label shown in the dashboard header |
| `enable_prometheus` | No | `false` | Generate and push a `metrics.prom` file for Prometheus |

### Required Permissions

The workflow requires the following repository permissions:

| Permission | Purpose |
| --- | --- |
| `contents: write` | Push reports to the report branch |
| `packages: read` | Pull test container images from GHCR |
| `issues: write` | Create and comment on GitHub issues |
| `pages: read` | Fetch the GitHub Pages URL for dashboard deep links in issues |

### Steps

#### 1. Run tests
The orchestrator container (`ghcr.io/vliz-be-opsci/grmp-framework:main`) is run with the selected config directory mounted at `/config`. The orchestrator reads all `.yaml` and `.yml` files under that directory, runs the configured test containers, and writes a `combined_report.xml` to the `reports/` directory.

The following GitHub Actions context variables are passed to the orchestrator so it can construct full provenance URLs:

- `GITHUB_SERVER_URL`
- `GITHUB_REPOSITORY`
- `GITHUB_SHA`
- `CONFIG_DIRECTORY`

#### 2. Push report to branch
The `combined_report.xml` is committed to the report branch as `report.xml`. If the branch does not yet exist, it is created as an orphan branch. The latest `dashboard/index.html` from the source branch is also copied across on each run, keeping the dashboard up to date. The commit message includes the run number, attempt number, branch name, and short commit SHA.

#### 3. Parse report and generate run metadata
The combined report is parsed to extract per-suite statistics (total, passed, failed, errored, skipped, pass rate, duration). A `new_run.json` file is generated containing the full run metadata including run number, run attempt, source branch, triggering actor, and summary statistics.

#### 4. Create or update issues for failing test suites
For each test suite in the report that has `create-issue: true` set in its configuration, the workflow checks for existing open GitHub issues. Issues are identified by a title of the form `[GRMP] {suite_name} #{config_hash}` where the hash is a stable 8-character SHA-256 of the suite's input config properties (provenance and `create-issue` excluded from the hash).

- If the suite is **failing or erroring** and no open issue exists → a new issue is created with a properties table and a test case summary table, labelled `grmp-test-failure`
- If the suite is **failing or erroring** and an open issue exists → a comment is added with the current run's results
- If the suite is **passing** and an open issue exists → a comment is added noting recovery, suggesting the issue can be closed manually

Each issue comment includes a link to the GitHub Actions run and, if GitHub Pages is configured, a deep link to the dashboard pre-loaded with the corresponding run (`#run=<commit_sha>`). The GitHub Pages URL is fetched automatically from the GitHub API — no manual configuration is needed. This step runs with `continue-on-error: true` so a failure here does not block the remaining steps.

> **Note:** The `grmp-test-failure` label must exist in the repository before the first issue creation run.

#### 5. Update runs.json
The `new_run.json` metadata is prepended to `runs.json` on the report branch, which maintains a history of the most recent 50 runs. This file is read by the dashboard to populate the run selector dropdown. Reruns of the same workflow run are stored as separate entries distinguished by the `run_attempt` field.

#### 6. Generate and commit config.json
A `config.json` is written to the report branch containing the `reportUrl`, `runsUrl`, and `label` fields needed by the dashboard to locate its data sources. This file is updated on every run so the dashboard always points to the correct branch.

#### 7. Generate and commit metrics.prom (optional)
When `enable_prometheus` is set to `true`, a `metrics.prom` file is generated in Prometheus textfile collector format and committed to the report branch. The file contains per-suite metrics with `testsuite`, `run`, `attempt`, and `branch` labels:

| Metric | Description |
| --- | --- |
| `grmp_testsuite_tests_total` | Total test cases in the suite |
| `grmp_testsuite_passed_total` | Passing test cases |
| `grmp_testsuite_failed_total` | Failing test cases |
| `grmp_testsuite_errors_total` | Errored test cases |
| `grmp_testsuite_skipped_total` | Skipped test cases |
| `grmp_testsuite_pass_rate` | Pass rate between 0 and 1 |
| `grmp_testsuite_duration_seconds` | Suite execution time in seconds |
| `grmp_run_timestamp_seconds` | Unix timestamp of the run |

To ingest these metrics into Prometheus, use the Node Exporter textfile collector. A cron job on the monitoring server can periodically fetch the latest `metrics.prom` from the report branch:

```bash
*/15 * * * * curl -s https://raw.githubusercontent.com/{org}/{repo}/{report_branch}/metrics.prom \
    -o /path/to/node_exporter/textfiles/metrics.prom
```

---

## Dashboard

The dashboard (`dashboard/index.html`) is a self-contained static page that is pushed to the report branch on every workflow run. It reads `report.xml` and `runs.json` directly from the report branch via `raw.githubusercontent.com` and requires no server-side infrastructure.

Features:
- View the current report or any of the last 50 historical runs via a dropdown
- Run selector shows the run number, attempt (if a rerun), source branch, short SHA, pass rate, and date
- Deep-link to a specific run via the `#run=<commit_sha>` URL fragment — used by issue comments to link directly to the relevant run
- Filter test suites by status (passed, failed, errored, skipped)
- Filter test cases by status
- Search test suites by name
- Filter by suite property and value
- Clickable provenance links when running via GitHub Actions

If GitHub Pages is enabled for the repository, the dashboard is served at `https://{owner}.github.io/{repo}` (or a custom domain if configured).

---

## Adding a New Configuration Directory

To add a new set of tests:

1. Create a new top-level directory in the repository (e.g. `my-new-checks/`)
2. Add one or more `.yaml` config files following the [grmp-framework configuration format](https://github.com/vliz-be-opsci/grmp-framework)
3. Trigger the workflow with `config_directory` set to your new directory name

No changes to the workflow itself are needed.

---

## Secrets

If any test configuration uses the `!secret` YAML tag to reference sensitive values, the corresponding `SECRET_*` environment variables must be passed to the orchestrator container. This requires adding them to the `Run tests` step in the workflow:

```yaml
-e SECRET_MY_API_KEY="${{ secrets.MY_API_KEY }}" \
```

See the [grmp-framework documentation](https://github.com/vliz-be-opsci/grmp-framework) for full details on how secrets are handled.