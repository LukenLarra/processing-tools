# BDD Workflow Guide

## Overview

`processing-tools` provides a **centralized, reusable GitHub Actions workflow** for running BDD tests across multiple services. Any repository that wants to run BDD tests only needs to call this workflow with a few parameters — all the setup, execution, and reporting is handled here.

The workflow delegates test execution to the **[shepherd_bdd](https://github.com/LukenLarra/shepherd_bdd)** framework, which provides three composable actions:

| Action | Purpose |
|---|---|
| `shepherd_bdd/actions/discovery` | Scans feature files and generates a parallel execution matrix |
| `shepherd_bdd/actions/main` | Sets up the Python environment and runs the BDD tests |
| `shepherd_bdd/actions/publish-reports` | Merges JUnit XML reports and publishes them to GitHub Checks |

---

## How to use it

From any service repository, create a workflow file that calls the centralized workflow:

```yaml
# .github/workflows/bdd.yaml (in your service repo)
name: BDD Tests

on:
  push:
    branches: [main, master]
  pull_request:

jobs:
  bdd:
    uses: LukenLarra/processing-tools/.github/workflows/bdd.yaml@master
    with:
      service: your-service-name
```

The `service` parameter must match the name used in `insights-behavioral-spec` for the feature files (`features/<service>/`) and the config file (`bdd-configs/<service>-framework.yml`).

### Available inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `service` | Yes | — | Name of the service to test |
| `use_kafka` | No | `false` | Start a Kafka container |
| `use_postgres` | No | `false` | Start a PostgreSQL container |
| `postgres_version` | No | `13.9` | PostgreSQL version |
| `use_minio` | No | `false` | Start a MinIO container |
| `use_pushgateway` | No | `false` | Start a Prometheus Pushgateway container |
| `use_mock-oauth2-server` | No | `false` | Start a mock OAuth2 server container |
| `go_version` | No | `1.24` | Go version used to build the service binary |
| `install_service` | No | `false` | Install the service as a Python package via `pip install` |
| `bdd_config` | No | `insights-behavioral-spec/bdd-configs/framework.yml` | Path to the BDD framework config file |
| `parallel_execution` | No | `false` | Run each feature file as an independent parallel job |
| `max_parallel` | No | `256` | Maximum number of simultaneous parallel jobs |

---

## Workflow structure

The workflow has four jobs:

```
discover_features (if parallel_execution)  ──┐
                                             ├──► bdd (matrix or single) ──► publish_report
build_binary                               ──┘
```

### Job 1 — `discover_features`

Only runs when `parallel_execution: true`. Calls `shepherd_bdd/actions/discovery` to scan the feature files defined in the BDD config and produce a JSON matrix used by the `bdd` job.

```yaml
- name: Run discovery
  id: discovery
  uses: LukenLarra/shepherd_bdd/actions/discovery@main
  with:
    bdd_config: "${{ github.workspace }}/framework.yml"
```

Outputs:
- `matrix` — JSON array of feature file paths
- `has_features` — `true` if at least one feature file was found

### Job 2 — `build_binary`

Compiles the service binary once and uploads the entire workspace as an artifact. The `bdd` job downloads this artifact, avoiding redundant builds when running in parallel.

Build lookup order:
1. `build.sh` — runs it if present
2. `Makefile` with a `build` target — runs `make build`
3. No build step — skips silently (e.g. for Python services)

### Job 3 — `bdd`

The main test execution job. Runs as a matrix (one job per feature file) when `parallel_execution: true`, or as a single job otherwise.

**Steps:**

```yaml
# 1. Restore the compiled workspace from build_binary
- name: Download workspace build
  uses: actions/download-artifact@v4
  with:
    name: ${{ needs.build_binary.outputs.artifact_name }}
    path: .

- name: Restore permissions and path
  run: |
    chmod -R +x * || true
    echo "$GITHUB_WORKSPACE" >> $GITHUB_PATH

# 2. Check out the BDD spec repository alongside the service
- name: Checkout insights-behavioral-spec
  uses: actions/checkout@v4
  with:
    repository: LukenLarra/insights-behavioral-spec
    ref: main
    path: insights-behavioral-spec

# 3. Install kcat if Kafka is needed
- name: Install kcat
  if: ${{ inputs.use_kafka }}
  run: sudo apt-get install -y kafkacat

# 4. Symlink the framework config and all spec resources into the workspace root
- name: Link framework config and resources
  run: |
    ln -sf "${{ github.workspace }}/insights-behavioral-spec/bdd-configs/${{ inputs.service }}-framework.yml" framework.yml
    ln -sf "${{ github.workspace }}/insights-behavioral-spec/features/${{ inputs.service }}" features
    for item in insights-behavioral-spec/*/; do
      dir_name=$(basename "$item")
      if [ "$dir_name" = "bdd-configs" ] || [ "$dir_name" = "features" ]; then continue; fi
      if [ ! -e "$dir_name" ]; then
        ln -sf "${{ github.workspace }}/insights-behavioral-spec/$dir_name" "$dir_name"
      fi
    done

# 5. Map service hostnames to localhost (required by some feature files)
- name: Add hosts entries
  if: ${{ inputs.use_pushgateway || inputs.use_mock-oauth2-server }}
  run: |
    if ${{ inputs.use_pushgateway }}; then
      echo "127.0.0.1 pushgateway" | sudo tee -a /etc/hosts
    fi
    if ${{ inputs.use_mock-oauth2-server }}; then
      echo "127.0.0.1 mock-oauth2-server" | sudo tee -a /etc/hosts
    fi

# 6. Run the tests via shepherd_bdd
- name: Run BDD Framework
  uses: LukenLarra/shepherd_bdd/actions/main@main
  with:
    service: ${{ inputs.service }}
    bdd_config: "${{ github.workspace }}/framework.yml"
    test_requirements: "insights-behavioral-spec/requirements.txt"
    service_package: ${{ inputs.install_service == true && github.workspace || '' }}
    feature_file: ${{ matrix.feature_file }}
    artifact_name_suffix: ${{ inputs.parallel_execution == true && format('-job-{0}', strategy.job-index) || '' }}
```

### Job 4 — `publish_report`

Always runs after `bdd` (even on failure) to collect and publish test results.

```yaml
- name: Download partial reports
  uses: actions/download-artifact@v4
  with:
    pattern: bdd-test-reports-${{ inputs.service }}-*
    path: reports/junit
    merge-multiple: true

- name: Publish merged test results
  uses: LukenLarra/shepherd_bdd/actions/publish-reports@main
  with:
    report_path: "reports/junit/*.xml"
```

---

## Infrastructure services

All infrastructure services are optional and controlled via inputs. When disabled, the image field is set to an empty string so GitHub Actions skips the container entirely.

Each service is exposed on `localhost` via explicit port mapping:

| Service | Input | Port |
|---|---|---|
| Kafka | `use_kafka` | 9092 |
| PostgreSQL | `use_postgres` | 5432 |
| MinIO | `use_minio` | 9000 |
| Pushgateway | `use_pushgateway` | 9091 |
| Mock OAuth2 Server | `use_mock-oauth2-server` | 8081 |

Kafka also requires `KAFKA_ADVERTISED_HOST_NAME: localhost` and a health check, since the image defaults to announcing itself as `kafka` (suitable for docker-compose but not for GitHub Actions runners).

---

## Design decisions

### Symlinks instead of copies

The `Link framework config and resources` step creates symlinks rather than copying files. This keeps `bdd_framework.py`'s `root_path` at the workspace root so all relative paths in the config resolve correctly, without duplicating content. Directories that already exist in the workspace (e.g. from the build artifact) are skipped to avoid conflicts.

### Separate `build_binary` job

Compiling once and sharing the artifact means parallel `bdd` matrix jobs don't each rebuild the binary. It also keeps the build step decoupled from test infrastructure setup.

### `/etc/hosts` entries for named services

Some feature files reference services by hostname (`pushgateway:9091`, `mock-oauth2-server:8081`). These hostnames are not resolvable on GitHub Actions runners, so entries are added to `/etc/hosts` to map them to `localhost` without modifying the feature files.
