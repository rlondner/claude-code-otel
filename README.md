# Claude Code Observability Stack

[![GitHub](https://img.shields.io/badge/GitHub-ColeMurray%2Fclaude--code--otel-blue?logo=github)](https://github.com/ColeMurray/claude-code-otel)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/Docker-Ready-blue?logo=docker)](docker-compose.yml)

A comprehensive observability solution for monitoring Claude Code usage, performance, and costs. This setup implements the recommendations from the [Claude Code Observability Documentation](CLAUDE_OBSERVABILITY.md) to provide deep insights into AI-assisted development workflows.

> **This is a fork of [ColeMurray/claude-code-otel](https://github.com/ColeMurray/claude-code-otel).** See the [Modifications from Upstream](#-modifications-from-upstream) section for a detailed description of what has changed.

## 🔀 Modifications from Upstream

This fork extends [ColeMurray/claude-code-otel](https://github.com/ColeMurray/claude-code-otel) with the following changes:

### 1. `last_over_time` instead of `increase` for cumulative counters

Several PromQL expressions in the unified dashboard were changed from `increase()` to `last_over_time()` for token usage metrics (total tokens, output tokens, cache read tokens, cache creation tokens).

**Why this matters:** `increase()` is a projection-based function — it extrapolates across the selected time range and can produce fractional or inflated values when metrics are sparse or scraped at irregular intervals (as is common with Claude Code sessions). `last_over_time()` returns the most recent actual value within the range, which is the correct semantics for these monotonically-increasing counters: you want the true cumulative total, not a rate-based estimate.

Affected panels in `claude-code-dashboard-unified.json`:
- Total Token Usage
- Output Tokens
- Cache Read Tokens
- Cache Creation Tokens

Example — before:
```promql
sum by (service_instance_id)(
  increase(claude_code_token_usage_tokens_total[$__range])
)
```

After:
```promql
sum by (service_instance_id)(
  last_over_time(claude_code_token_usage_tokens_total[$__range])
)
```

---

### 2. OpenLIT added to the observability stack

The `docker-compose-openlit.yml` file adds [OpenLIT](https://openlit.io/) — an open-source AI observability platform — alongside the existing Prometheus/Loki/Grafana stack.

**New components:**

| Service | Purpose | Port(s) |
|---------|---------|---------|
| **OpenLIT** | AI observability UI + built-in OTLP receiver | 3000 (UI), 4317 (gRPC), 4318 (HTTP) |
| **ClickHouse** | Column-store database backend for OpenLIT | 9000 (native), 8123 (HTTP) |

**Architecture with OpenLIT:**
```
Claude Code → OpenLIT (OTLP 4317/4318) → ClickHouse
                    ↓
         OTel Collector → ClickHouse
                    ↓
              Prometheus (scrape :8889)
                    ↓
              Grafana (port 3001)
```

**Port changes** (compared to the base repo, to avoid conflicts with OpenLIT on 3000):
- Grafana is now on **port 3001** (was 3000)
- The standalone OTel Collector OTLP gRPC receiver is on **port 4319** (was 4317)
- The standalone OTel Collector OTLP HTTP receiver is on **port 4320** (was 4318)

The `assets/otel-collector-config.yaml` is configured to export traces, logs, and metrics to ClickHouse, and also performs token normalization via the `transform` processor — mapping Claude-specific attributes (`input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_creation_tokens`) to the standard OpenTelemetry `gen_ai.*` keys that OpenLIT understands for model pricing lookups.

To start the stack with OpenLIT:
```bash
docker compose -f docker-compose-openlit.yml up -d
```

OpenLIT UI is then available at **http://localhost:3000**, Grafana at **http://localhost:3001**.

---

### 3. Additional Grafana dashboards

Several new dashboard JSON files have been added under the root directory and are automatically provisioned into Grafana:

| File | Dashboard Title | Description |
|------|----------------|-------------|
| `claude-code-dashboard-unified.json` | Claude Code Unified | Comprehensive all-in-one view: overview, costs, token usage, tool performance, errors, activity |
| `claude-code-dashboard-metrics.json` | Claude Code - Metrics | Core metrics view focused on API usage and token counts |
| `claude-code-dashboard-metrics-v2.json` | Claude Code - Metrics v2 | Extended metrics with additional breakdowns and panels |
| `claude-code-dashboard-summary.json` | Claude Code - Daily/Weekly Summary | Aggregated daily and weekly productivity summary |
| `claude-code-dashboard-economics.json` | Claude Code - Engineering Economics | Cost/value analysis using engineering ROI model |
| `claude-code-dashboard-economics-max200.json` | Claude Code - Engineering Economics (Max 200 Plan) | Economics dashboard calibrated for the Max 200 subscription plan |
| `claude-code-dashboard-economics-pro20.json` | Claude Code - Engineering Economics (Pro Plan) | Economics dashboard calibrated for the Pro ($20) subscription plan |

These are all provisioned automatically via `grafana-dashboards.yml` — no manual import needed.

---

## 📸 Dashboard Screenshots

### 💰 Cost & Usage Analysis
Track spending across different Claude models with detailed breakdowns of costs, API requests, and token usage patterns.

<img src="docs/images/cost-usage-analytics.png" alt="Cost & Usage Analysis Dashboard" width="800">

*Features: Model cost comparison, API request tracking, token usage breakdown by type*

### 📊 User Activity & Productivity 
Monitor development productivity with comprehensive session analytics, tool usage patterns, and code change metrics.

<img src="docs/images/user-activity.png" alt="User Activity & Productivity Dashboard" width="800">

*Features: Session tracking, tool performance metrics, code productivity insights*

## 🎯 Features

### 📊 **Comprehensive Monitoring**
- **Cost Analysis**: Track usage costs by model, user, and time periods
- **User Analytics**: Daily/Weekly/Monthly Active Users (DAU/WAU/MAU)
- **Tool Usage**: Monitor which Claude Code tools are used most frequently
- **Performance Metrics**: API latency, success rates, and bottleneck identification
- **Productivity Insights**: Lines of code changes, commits, and pull requests

### 📊 **Enhanced Analytics**
- **API Request Tracking**: Monitor actual request counts by model version
- **Token Efficiency**: Track cost-per-token across different models
- **Session Analytics**: Comprehensive session and productivity tracking
- **Real-time Monitoring**: Live dashboards with 30-second refresh rates

### 📈 **Rich Dashboards**
- **Executive Overview**: High-level KPIs and trends
- **Cost Management**: Detailed cost breakdowns and projections
- **Tool Performance**: Success rates and execution times
- **User Activity**: Productivity and engagement metrics
- **Error Analysis**: Comprehensive error tracking and investigation

## 🏗️ Architecture

This repo ships two Docker Compose configurations:

### Base stack (`docker-compose.yml`)
```
Claude Code → OTel Collector (4317/4318) → Prometheus + Loki
                                                    ↓
                                             Grafana :3000
```

### Extended stack with OpenLIT (`docker-compose-openlit.yml`)
```
Claude Code → OpenLIT (4317/4318) → ClickHouse
                    ↓
         OTel Collector (4319/4320) → ClickHouse
                    ↓
         Prometheus (scrape :8889) + Loki
                    ↓
             Grafana :3001
```

### Components

#### Base stack

| Service | Purpose | Port | UI |
|---------|---------|------|----| 
| **OpenTelemetry Collector** | Metrics/logs ingestion | 4317 (gRPC), 4318 (HTTP) | - |
| **Prometheus** | Metrics storage & querying | 9090 | http://localhost:9090 |
| **Loki** | Log aggregation & storage | 3100 | - |
| **Grafana** | Dashboards & visualization | 3000 | http://localhost:3000 |

#### Extended stack (with OpenLIT)

| Service | Purpose | Port | UI |
|---------|---------|------|----| 
| **OpenLIT** | AI observability UI + OTLP receiver | 3000 (UI), 4317 (gRPC), 4318 (HTTP) | http://localhost:3000 |
| **ClickHouse** | Column-store backend for OpenLIT | 9000 (native), 8123 (HTTP) | - |
| **OpenTelemetry Collector** | Additional ingestion + token normalization | 4319 (gRPC), 4320 (HTTP) | - |
| **Prometheus** | Metrics storage & querying | 9090 | http://localhost:9090 |
| **Loki** | Log aggregation & storage | 3100 | - |
| **Grafana** | Dashboards & visualization | 3001 | http://localhost:3001 |

## 🚀 Quick Start

### 1. Start the Stack

**Base stack (Prometheus + Loki + Grafana):**
```bash
make up
# or
docker compose up -d
```

**Extended stack (adds OpenLIT + ClickHouse):**
```bash
make up2
# or
docker compose -f docker-compose-openlit.yml up -d
```

### 2. Configure Claude Code

#### Option A — `settings.json` (recommended, persistent)

Add the following to your Claude Code `settings.json` (typically `~/.claude/settings.json`). This is the preferred approach because the settings persist across terminal sessions and don't need to be re-exported every time.

```json
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "CLAUDE_CODE_ENHANCED_TELEMETRY_BETA": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_TRACES_EXPORTER": "otlp",
    "OTEL_LOG_USER_PROMPTS": "1",
    "OTEL_LOG_TOOL_CONTENT": "1",
    "OTEL_EXPORTER_OTLP_PROTOCOL": "http/json",
    "OTEL_EXPORTER_OTLP_METRICS_ENDPOINT": "http://localhost:4320/v1/metrics",
    "OTEL_EXPORTER_OTLP_LOGS_ENDPOINT": "http://localhost:4320/v1/logs",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "http://localhost:4318",
    "OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE": "cumulative"
  }
}
```

> **Note on endpoints:** The settings above target the extended stack (OpenLIT on ports 4318/4320 HTTP/JSON). For the base stack, change the endpoints to `http://localhost:4317` and use `"OTEL_EXPORTER_OTLP_PROTOCOL": "grpc"`.

> **`OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE": "cumulative"`** — ensures token and cost counters are reported as ever-increasing totals rather than per-export deltas, which is required for `last_over_time` queries to return correct values in Grafana.

#### Option B — environment variables (per-session)

```bash
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_TRACES_EXPORTER=otlp
export OTEL_LOG_USER_PROMPTS=1
export OTEL_LOG_TOOL_CONTENT=1
export OTEL_EXPORTER_OTLP_PROTOCOL=http/json
export OTEL_EXPORTER_OTLP_METRICS_ENDPOINT=http://localhost:4320/v1/metrics
export OTEL_EXPORTER_OTLP_LOGS_ENDPOINT=http://localhost:4320/v1/logs
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
export OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=cumulative

# Run Claude Code
claude
```

### 3. Access Dashboards

**Base stack:**
- **Grafana**: http://localhost:3000 (admin/admin)
- **Prometheus**: http://localhost:9090

**Extended stack:**
- **OpenLIT**: http://localhost:3000
- **Grafana**: http://localhost:3001 (admin/admin)
- **Prometheus**: http://localhost:9090

> 🖼️ **Visual Guide**: Check out the [Dashboard Screenshots](#-dashboard-screenshots) to see what your dashboards will look like!

## 📊 Available Metrics

Based on the [Claude Code Observability Documentation](CLAUDE_OBSERVABILITY.md), this stack monitors:

### Core Metrics
- `claude_code.session.count` - CLI sessions started
- `claude_code.lines_of_code.count` - Lines of code modified (added/removed)
- `claude_code.pull_request.count` - Pull requests created
- `claude_code.commit.count` - Git commits created
- `claude_code.cost.usage` - Cost of sessions by model
- `claude_code.token.usage` - Token usage (input/output/cache/creation)
- `claude_code.code_edit_tool.decision` - Tool permission decisions

### Event Data
- `claude_code.user_prompt` - User prompt submissions
- `claude_code.tool_result` - Tool execution results and timings
- `claude_code.api_request` - API requests with duration and tokens
- `claude_code.api_error` - API errors with status codes
- `claude_code.tool_decision` - Tool permission decisions

## 🔍 Usage Analysis

### Real-time Dashboard Analysis

Access comprehensive analytics through the Grafana dashboard at http://localhost:3000 (base stack) or http://localhost:3001 (extended stack with OpenLIT):

- **Cost Analysis**: Real-time cost tracking with model breakdowns
- **Request Monitoring**: API request counts and patterns by model
- **Token Efficiency**: Track token usage and cost-per-token metrics
- **Tool Performance**: Success rates and execution time analysis
- **Session Analytics**: User activity and productivity insights

### Key Metrics Available
- Total and per-model costs with trending
- API request counts independent of cost variations
- Token usage breakdown (input/output/cache/creation)
- Tool usage patterns and success rates
- Session activity and code productivity metrics

## 📊 Key Dashboard Features

> 💡 **See [Dashboard Screenshots](#-dashboard-screenshots) above for visual examples**

### 💰 Cost & Usage Analysis
- **Cost by Model**: Track spending across different Claude models
- **API Request Tracking**: Monitor actual request counts by model version  
- **Token Usage Breakdown**: Detailed analysis by token type (input/output/cache)

### 🔧 Tool Performance
- **Usage Patterns**: Most frequently used Claude Code tools
- **Success Rates**: Tool execution success percentages
- **Performance Metrics**: Average execution times and bottleneck identification

### ⚡ Real-time Monitoring
- **Live Metrics**: 30-second refresh rate for current activity
- **Session Tracking**: Active sessions and productivity metrics
- **Error Analysis**: API errors and troubleshooting information

## 📋 Available Dashboards

Seven dashboards are provisioned automatically into Grafana. The **Unified** dashboard is the recommended starting point; the others provide specialized views.

### Claude Code Unified (`claude-code-dashboard-unified.json`)

The all-in-one dashboard, organized into these sections:

**📊 Overview** — Active sessions, cumulative cost, total tokens, lines of code changed

**💰 Cost & Usage Analysis** — Cost trends by model, API request count tracking, token usage breakdown by type (input/output/cache read/cache creation). Token counts use `last_over_time` for accurate cumulative values — see [Modifications from Upstream](#-modifications-from-upstream).

**🔧 Tool Usage & Performance** — Tool frequency rankings, success rates, execution time distributions, bottleneck identification

**⚡ Performance & Errors** — API latency by model, error rate tracking, P50/P95/P99 response times

**📝 User Activity & Productivity** — Lines of code changed, commits created, pull requests opened

**🔍 Event Logs** — Real-time tool execution events and API errors via Loki

---

### Claude Code Metrics (`claude-code-dashboard-metrics.json`)
Core metrics view: API usage counts, token totals, session counts. Lightweight and fast to load.

### Claude Code Metrics v2 (`claude-code-dashboard-metrics-v2.json`)
Extended metrics with additional dimension breakdowns and more panels.

### Claude Code Daily/Weekly Summary (`claude-code-dashboard-summary.json`)
Aggregated view of daily and weekly activity — useful for team standup reports and weekly reviews.

### Engineering Economics dashboards

Three variants of a cost/ROI analysis dashboard that model the economic value of Claude Code assistance:

| Dashboard | Plan | Monthly cost cap used |
|-----------|------|-----------------------|
| `claude-code-dashboard-economics.json` | Generic | Configurable |
| `claude-code-dashboard-economics-max200.json` | Max 200 | $200/month |
| `claude-code-dashboard-economics-pro20.json` | Pro | $20/month |

These dashboards calculate estimated engineering value (time saved × developer hourly rate) against subscription cost to produce an ROI figure.

## 🔧 Advanced Configuration

### Environment Variables

Key configuration options (see [CLAUDE_OBSERVABILITY.md](CLAUDE_OBSERVABILITY.md) for complete reference):

```bash
# Core telemetry
CLAUDE_CODE_ENABLE_TELEMETRY=1

# Exporter configuration
OTEL_METRICS_EXPORTER=otlp,prometheus    # Multiple exporters
OTEL_LOGS_EXPORTER=otlp

# Protocol and endpoints
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer token"

# Export intervals
OTEL_METRIC_EXPORT_INTERVAL=60000        # 1 minute (production)
OTEL_LOGS_EXPORT_INTERVAL=5000           # 5 seconds

# Privacy controls
OTEL_LOG_USER_PROMPTS=1                   # Enable prompt content logging

# Cardinality control
OTEL_METRICS_INCLUDE_SESSION_ID=true
OTEL_METRICS_INCLUDE_VERSION=false
OTEL_METRICS_INCLUDE_ACCOUNT_UUID=true
```

### Collector Configuration

The OpenTelemetry collector is configured with:
- **Processors**: Resource enrichment and event filtering
- **Multiple Pipelines**: Separate routing for metrics and different event types
- **Metric Relabeling**: Cardinality control for better performance

### Backend Considerations

Following the documentation recommendations:

- **Metrics Backend**: Prometheus (time series) — scraped from the OTel Collector's Prometheus exporter on `:8889`
- **Events/Logs Backend**: Loki (log aggregation) with JSON parsing
- **AI Observability Backend**: ClickHouse (columnar store) — used by OpenLIT in the extended stack for traces, logs, and metrics with model-level cost attribution
- **Cardinality Management**: Configurable attribute inclusion
- **Retention**: Configure based on your analysis needs (ClickHouse TTL defaults to 730 hours / ~30 days)

## 🛠️ Management Commands

```bash
# Stack management
make up                    # Start all services
make down                  # Stop all services  
make restart              # Restart services
make clean                # Clean up containers and volumes

# Monitoring
make logs                 # View all logs
make logs-collector       # View collector logs only
make status              # Show service status

# Validation
make validate-config     # Validate all configs
make setup-claude       # Show Claude Code setup instructions
```

## 🎯 Use Cases

### For Engineering Teams
- **Cost Management**: Track AI assistance costs by team/project
- **Productivity Measurement**: Quantify development velocity improvements
- **Tool Adoption**: Understand which Claude Code features drive value
- **Performance Optimization**: Identify and resolve usage bottlenecks

### For Platform Teams
- **Capacity Planning**: Predict infrastructure needs based on usage growth
- **SLA Monitoring**: Track API performance and availability
- **Security**: Monitor unusual usage patterns
- **Resource Optimization**: Optimize token usage and reduce costs

### For Management
- **ROI Analysis**: Measure productivity gains from AI assistance
- **Usage Insights**: Understand adoption patterns across teams
- **Cost Control**: Monitor and optimize AI assistance spending
- **Strategic Planning**: Data-driven decisions on AI tool investments

## 🔒 Security & Privacy

- **User Privacy**: Prompt content logging is disabled by default
- **Data Isolation**: All data stays within your infrastructure
- **Access Control**: Configure Grafana authentication as needed
- **Audit Trail**: Complete logging of all tool usage and decisions

## 📚 Resources

- [Claude Code Observability Documentation](CLAUDE_OBSERVABILITY.md) - Complete reference
- [Upstream repository: ColeMurray/claude-code-otel](https://github.com/ColeMurray/claude-code-otel) - Original project this fork is based on
- [OpenLIT Documentation](https://docs.openlit.io/) - AI observability platform
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/) - OTel specification
- [Prometheus Documentation](https://prometheus.io/docs/) - Metrics and alerting
- [Grafana Documentation](https://grafana.com/docs/) - Dashboards and visualization
- [Loki Documentation](https://grafana.com/docs/loki/) - Log aggregation
- [ClickHouse Documentation](https://clickhouse.com/docs/) - Column-store database used by OpenLIT

## 🤝 Contributing

This observability stack implements the patterns and recommendations from the official Claude Code documentation. To contribute:

1. Follow the metric naming conventions in the documentation
2. Update dashboards to reflect new data sources and metrics
3. Test configurations before submitting changes
4. Ensure all sensitive information is excluded from commits
5. Update documentation for any new features or configuration changes

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- Forked from [ColeMurray/claude-code-otel](https://github.com/ColeMurray/claude-code-otel) — the original observability stack this project builds on
- Built following the [Claude Code Observability Documentation](CLAUDE_OBSERVABILITY.md)
- Uses OpenTelemetry standards for metrics and events
- Extended with [OpenLIT](https://openlit.io/) for AI-native observability with model cost attribution
- Implements industry best practices for observability stack architecture