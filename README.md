# gascity-otel

OpenTelemetry observability stack for [Gas City](https://github.com/gastownhall/gascity) — pre-configured VictoriaMetrics + VictoriaLogs + Grafana with dashboards.

## Stack

| Service | Port | Purpose |
|---------|------|---------|
| VictoriaMetrics | 8428 | Metrics storage (Prometheus-compatible) |
| VictoriaLogs | 9428 | Log storage (OTLP insert) |
| Grafana | 9429 | Dashboards and exploration |

## Quick Start

```bash
# Start the stack
docker compose up -d

# Set env vars for gc + bd + Claude Code telemetry
source setup.sh

# Verify — triggers bd calls that produce metrics + logs
gc status
```

Open Grafana at [http://localhost:9429](http://localhost:9429) (admin/admin). The **Gas City** folder contains three pre-built dashboards.

## Environment Variables

`source setup.sh` exports all required variables. You can also add them to `~/.zshrc` or `~/.bashrc`.

### Gas City SDK

| Variable | Value | Purpose |
|----------|-------|---------|
| `GC_OTEL_METRICS_URL` | `http://localhost:8428/opentelemetry/api/v1/push` | OTLP metrics push endpoint |
| `GC_OTEL_LOGS_URL` | `http://localhost:9428/insert/opentelemetry/v1/logs` | OTLP logs insert endpoint |
| `BD_OTEL_METRICS_URL` | (same as GC) | Auto-propagated to bd subprocesses |
| `BD_OTEL_LOGS_URL` | (same as GC) | Auto-propagated to bd subprocesses |

### Claude Code (optional)

| Variable | Value | Purpose |
|----------|-------|---------|
| `CLAUDE_CODE_ENABLE_TELEMETRY` | `1` | Enable OTLP export |
| `OTEL_METRICS_EXPORTER` | `otlp` | Export metrics via OTLP |
| `OTEL_LOGS_EXPORTER` | `otlp` | Export logs via OTLP |
| `OTEL_EXPORTER_OTLP_METRICS_PROTOCOL` | `http/protobuf` | Required by VictoriaMetrics |
| `OTEL_EXPORTER_OTLP_LOGS_PROTOCOL` | `http/protobuf` | Required by VictoriaLogs |

### Privacy-Sensitive Variables

These are set by `setup.sh` but you may want to disable them in shared environments:

| Variable | Default | Risk |
|----------|---------|------|
| `GC_LOG_BD_OUTPUT` | unset | Captures bd stdout/stderr — may contain code, secrets, or PII |
| `OTEL_LOG_TOOL_CONTENT` | `true` | Captures Claude tool output (file contents, command results) |
| `OTEL_LOG_USER_PROMPTS` | `true` | Captures user messages sent to Claude |

## Dashboards

### Agent Operations

Agent lifecycle, bd call performance, and session activity.

- **Stats**: bead store health, agent starts/crashes/quarantines (1h)
- **Agent lifecycle**: starts by agent, stops by reason, idle kills
- **bd calls**: call rate and error rate by subcommand, latency p50/p95/p99
- **Sessions**: nudge rate, sling dispatches, reconcile cycles
- **Controller**: config reloads by status, controller lifecycle events

### AI & Token Usage

Claude API usage via bd metrics.

- **Stats**: input/output tokens (1h), median API latency
- **Token rates**: input and output by model
- **Latency**: API p50/p95/p99
- **Efficiency**: output/input token ratio

### Beads Store Health

bd storage operations, errors, and lock contention.

- **Stats**: storage ops/errors (1h), DB retries, open issues
- **Operations**: op rate and error rate by type
- **Latency**: storage op p50/p95, lock wait p50/p95
- **Circuit breaker**: trip rate, rejection rate
- **Issues**: count by status

## Metric Reference

### gc SDK metrics

| Metric | Type | Labels |
|--------|------|--------|
| `gc_agent_starts_total` | counter | agent, status |
| `gc_agent_stops_total` | counter | agent, reason, status |
| `gc_agent_crashes_total` | counter | agent |
| `gc_agent_quarantines_total` | counter | agent |
| `gc_agent_idle_kills_total` | counter | agent |
| `gc_reconcile_cycles_total` | counter | started, stopped, skipped |
| `gc_session_nudges_total` | counter | target, status |
| `gc_config_reloads_total` | counter | status |
| `gc_controller_lifecycle_total` | counter | event |
| `gc_bd_calls_total` | counter | status, subcommand |
| `gc_sling_dispatches_total` | counter | target, target_type, method, status |
| `gc_bead_store_healthy` | gauge | city |
| `gc_bd_duration_ms` | histogram | status, subcommand |

### bd metrics

| Metric | Type | Labels |
|--------|------|--------|
| `bd_storage_operations_total` | counter | type |
| `bd_storage_operation_duration_ms` | histogram | type |
| `bd_storage_errors_total` | counter | type |
| `bd_db_retry_count_total` | counter | — |
| `bd_db_lock_wait_ms` | histogram | — |
| `bd_db_circuit_trips_total` | counter | — |
| `bd_db_circuit_rejected_total` | counter | — |
| `bd_issue_count` | gauge | status |
| `bd_ai_input_tokens_total` | counter | model |
| `bd_ai_output_tokens_total` | counter | model |
| `bd_ai_request_duration_ms` | histogram | model |

## Log Events

### gc SDK events (VictoriaLogs)

`agent.start`, `agent.stop`, `agent.crash`, `agent.quarantine`, `agent.idle_kill`, `reconcile.cycle`, `session.nudge`, `config.reload`, `controller.lifecycle`, `bd.call`, `sling.dispatch`, `bead_store.health`

### Claude Code events (when OTEL enabled)

`claude_code.api_request`, `claude_code.tool_result`, `claude_code.tool_decision`, `claude_code.user_prompt`

Resource attributes: `gc.agent`, `gc.rig`, `gc.city`

## Query UIs

After `source setup.sh`, these URLs are printed:

- **VictoriaMetrics**: http://localhost:8428/vmui — PromQL queries
- **VictoriaLogs**: http://localhost:9428/select/vmui — log search + live tailing
- **Grafana**: http://localhost:9429 — dashboards

## License

Apache 2.0
