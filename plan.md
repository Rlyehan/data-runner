# Data Runner - Workflow Orchestration Platform

## Architecture Overview

**Supervisor** (main control plane):

- REST API for workflow/worker management
- gRPC server for worker status updates and logs
- SQLite for state persistence
- AWS SDK for EC2 provisioning
- Git repo watcher for workflow definitions

**Worker Agent** (runs on EC2):

- gRPC client for supervisor communication
- Executes Polars (Python) or DuckDB (SQL) workflows
- Streams execution logs and status

**Workflow Structure** (Git repo):

```
workflows/
  data-pipeline-1/
    config.yaml          # worker_type: polars, instance_size: t3.medium
    workflow.py          # Polars code
  etl-job-2/
    config.yaml          # worker_type: duckdb, instance_size: t3.small
    workflow.sql         # DuckDB SQL
```

## Implementation Plan

### Phase 1: Project Setup & Core Models

- Initialize Rust workspace with `supervisor` and `worker-agent` crates
- Define gRPC proto files for worker<->supervisor communication (WorkerStatus, LogStream, HeartBeat)
- Set up SQLite schema (workflows, workers, executions, logs tables)
- Create core domain models (Workflow, Worker, Execution)

### Phase 2: Supervisor - REST API

- Implement REST API using `axum`:
  - `POST /workflows` - register workflow from git repo
  - `GET /workflows` - list all workflows
  - `POST /workflows/{id}/execute` - trigger execution
  - `GET /executions/{id}` - get execution status/logs
  - `GET /workers` - list active workers
- Add workflow config parser (parse `config.yaml`)

### Phase 3: Supervisor - Worker Provisioning

- AWS SDK integration for EC2:
  - Create EC2 instances based on worker type (Polars/DuckDB)
  - Use user-data scripts to bootstrap worker agent
  - Generate AMIs or use startup scripts to install dependencies
- Worker lifecycle management (provision, track, terminate)

### Phase 4: Worker Agent

- Build worker agent binary:
  - gRPC client to connect to supervisor on startup
  - Execute workflow code (shell out to `python` or `duckdb`)
  - Stream stdout/stderr as logs via gRPC
  - Report status (RUNNING, SUCCESS, FAILED)
  - Graceful shutdown after execution

### Phase 5: Git Integration

- Clone/pull workflow repo on supervisor startup
- Watch for changes (optional: webhook for auto-sync)
- Transfer workflow code to worker EC2 (via S3 or embedded in user-data)

### Phase 6: Testing & Polish

- Integration tests with LocalStack for AWS mocking
- End-to-end test: register workflow → execute → check logs
- Add proper error handling and retries
- CLI client for easy REST API interaction (optional)

## Key Files Structure

```
data-runner/
  Cargo.toml              # workspace
  proto/
    worker.proto          # gRPC definitions
  supervisor/
    src/
      main.rs
      api/                # REST endpoints
      grpc/               # gRPC server
      db/                 # SQLite models
      aws/                # EC2 provisioning
      workflow/           # workflow parsing
  worker-agent/
    src/
      main.rs
      executor/           # run Polars/DuckDB
      grpc/               # gRPC client
```

## Dependencies

- `tokio` - async runtime
- `axum` - REST API
- `tonic` - gRPC
- `sqlx` - SQLite with compile-time query checking
- `aws-sdk-ec2` - EC2 provisioning
- `serde` + `serde_yaml` - config parsing
- `git2` - git operations