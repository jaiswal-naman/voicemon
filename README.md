# VoiceMon 🎙️

**Production-grade observability for voice AI pipelines.**

VoiceMon implements the [4-Layer Voice Observability Framework](https://www.hamming.ai/blog/voice-agent-observability) to give you full-stack monitoring of voice agents built on LiveKit, Pipecat, or Vapi — from network jitter to task completion.

```
┌─────────────────────────────────────────────────────────────────────┐
│                          YOUR VOICE AGENT                          │
│              (LiveKit Agents / Pipecat / Vapi)                     │
├────────────────────────────┬────────────────────────────────────────┤
│      voicemon SDK          │   1-line integration                  │
│  ┌──────────────────────┐  │   instrument_livekit(session, col)    │
│  │  VoiceMonCollector   │──│──▶ Redis Streams ──▶ Worker           │
│  └──────────────────────┘  │       │                  │            │
│         │ OTel Spans       │       ▼                  ▼            │
│         ▼                  │  TimescaleDB      Prometheus          │
│   Jaeger/Tempo             │       │                  │            │
│                            │       ▼                  ▼            │
│                            │  Streamlit         Grafana            │
│                            │  (Analytics)       (Ops)              │
│                            │       │                  │            │
│                            │       └──────┬───────────┘            │
│                            │              ▼                        │
│                            │     Slack / PagerDuty                 │
│                            │     (Alert Routing)                   │
└────────────────────────────┴────────────────────────────────────────┘
```

## 4-Layer Observability Framework

| Layer | What it captures | VoiceMon metrics |
|-------|-----------------|------------------|
| **L1: Infrastructure** | Network, codec, transport | Jitter, packet loss, MOS score, bitrate |
| **L2: Execution** | STT → LLM → TTS pipeline | Per-stage latency, confidence, tokens, TTFT/TTFB |
| **L3: User Experience** | Perceived quality | E2E latency, interruptions, silence ratio, TTFW |
| **L4: Outcome** | Business results | Task success, CSAT, resolution type, cost |

## 📚 Documentation

- **[DOCUMENTATION_SUMMARY.md](DOCUMENTATION_SUMMARY.md)** — Quick reference guide with key concepts
- **[ARCHITECTURE.md](ARCHITECTURE.md)** — Complete High-Level and Low-Level Design (HLD/LLD)
- **[SYSTEM_DESIGN.md](SYSTEM_DESIGN.md)** — System design, data flows, and deployment patterns

## Quick Start

### Install

```bash
pip install voicemon                    # Core SDK
pip install voicemon[livekit]           # + LiveKit integration
pip install voicemon[pipecat]           # + Pipecat integration
pip install voicemon[vapi]              # + Vapi integration
pip install voicemon[dashboard]         # + Streamlit dashboard
pip install voicemon[all]               # Everything
```

### LiveKit Integration (3 lines)

```python
from livekit.agents import AgentSession
from voicemon import VoiceMonCollector
from voicemon.integrations.livekit import instrument_livekit

collector = VoiceMonCollector()
session = AgentSession(...)

# One-line instrumentation
session_id = instrument_livekit(session, collector, agent_id="my-agent")
```

### Pipecat Integration

```python
from pipecat.pipeline.pipeline import Pipeline
from voicemon import VoiceMonCollector
from voicemon.integrations.pipecat import VoiceMonPipecatObserver

collector = VoiceMonCollector()
observer = VoiceMonPipecatObserver(collector, agent_id="my-pipecat-agent")
pipeline = Pipeline([...], observers=[observer])
```

### Vapi Integration (Webhooks)

```python
from fastapi import FastAPI
from voicemon import VoiceMonCollector
from voicemon.integrations.vapi import create_vapi_router

app = FastAPI()
collector = VoiceMonCollector()
app.include_router(create_vapi_router(collector, webhook_secret="your-secret"))
```

### Vapi Integration (Polling)

```python
from voicemon.integrations.vapi import VapiClient

client = VapiClient(api_key="your-key", collector=collector)
session_ids = await client.poll_recent_calls(minutes=5)
```

## Infrastructure Setup

### Docker Compose (Recommended)

```bash
# Start full observability stack
docker compose up -d

# Initialize the database schema
docker compose exec timescale psql -U voicemon -d voicemon -f /schema/schema.sql

# Access dashboards
open http://localhost:3000    # Grafana (admin/voicemon)
open http://localhost:8501    # Streamlit Analytics
```

This starts:
- **TimescaleDB** — time-series + relational storage for sessions/turns/events
- **Redis** — stream-based event ingestion with consumer groups
- **Prometheus** — SLO metrics and alerting
- **Grafana** — operational dashboards with pre-built voice panels
- **Streamlit** — analytical dashboard with call replay + drift detection
- **Worker** — async Redis consumer for aggregation and alert evaluation

### Demo Simulator

Generate realistic voice call telemetry to see the dashboards in action:

```bash
# Console output (no infrastructure needed)
python -m tests.demo_simulator --sessions 50

# With Redis export (requires docker compose)
python -m tests.demo_simulator --sessions 100 --redis
```

## Configuration

VoiceMon uses environment variables or `VoiceMonConfig`:

```python
from voicemon.core.config import VoiceMonConfig

config = VoiceMonConfig(
    redis={"url": "redis://localhost:6379/0"},
    timescale={"dsn": "postgresql://voicemon:voicemon@localhost:5432/voicemon"},
    slack={"webhook_url": "https://hooks.slack.com/..."},
    pagerduty={"routing_key": "your-routing-key"},
)
```

### Industry-Standard Thresholds

| Metric | OK | Warning | Critical |
|--------|------|---------|----------|
| E2E Latency | < 800ms | < 1200ms | > 1800ms |
| STT Latency | < 300ms | < 500ms | > 1000ms |
| LLM TTFT | < 500ms | < 800ms | > 2000ms |
| TTS TTFB | < 200ms | < 300ms | > 800ms |
| STT Confidence | > 0.85 | > 0.7 | < 0.5 |

## Alert Rules

Alerts are defined in `alert_rules.yaml` with severity-based routing:

- **Info/Warning** → Slack `#voicemon-alerts`
- **Critical** → Slack + @oncall mention
- **P0** → Slack + PagerDuty incident

```yaml
rules:
  - name: e2e_latency_critical
    metric: e2e_latency_ms
    operator: ">"
    threshold: 1800
    severity: p0
    cooldown_minutes: 2
    notify: [slack, pagerduty]
```

## Architecture

```
Voice Agent (LiveKit/Pipecat/Vapi)
        │
        ▼
VoiceMonCollector (SDK)
    │         │
    ▼         ▼
  OTel    Exporters
  Spans      │
    │     ┌──┴──┐
    ▼     ▼     ▼
Jaeger  Redis  Prometheus
        Streams  /metrics
          │        │
          ▼        ▼
    ┌─────────┐  Grafana
    │ Worker  │  (Ops Dashboard)
    │ Process │
    └────┬────┘
         │
    ┌────┴────┐
    ▼         ▼
TimescaleDB  Alert Engine
    │         │
    ▼      ┌──┴──┐
Streamlit  ▼     ▼
(Analytics) Slack PagerDuty
```

### Key Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Primary DB | TimescaleDB | Voice data is relational (sessions→turns→events need JOINs) + native time-series |
| Ingestion | Redis Streams | Lightweight, consumer groups for horizontal scaling, at-least-once delivery |
| Metrics | Prometheus | Industry standard for SLOs, native histogram quantiles, Alertmanager |
| Ops Dashboard | Grafana | Universal, supports both Prometheus + TimescaleDB datasources |
| Analytics | Streamlit | Python-native, custom call replay + drift detection UX |
| Anomaly Detection | Z-score | Simple, no ML dependencies, catches latency spikes in real-time |

## Package Structure

```
voicemon/
├── core/
│   ├── models.py          # Pydantic v2 data models (4-layer framework)
│   ├── config.py           # Configuration with industry-standard thresholds
│   ├── collector.py        # Central telemetry hub
│   └── otel.py             # OpenTelemetry bridge with voice-aware spans
├── integrations/
│   ├── livekit.py          # LiveKit AgentSession event hooks
│   ├── pipecat.py          # Pipecat BaseObserver frame interception
│   └── vapi.py             # Vapi webhooks + REST client
├── exporters/
│   ├── base.py             # Exporter interface + ConsoleExporter
│   ├── redis.py            # Redis Streams with consumer groups
│   └── prometheus.py       # Prometheus histograms/counters/gauges
├── storage/
│   ├── schema.sql          # TimescaleDB schema with hypertables + aggregates
│   └── timescale.py        # Async TimescaleDB client
├── workers/
│   └── processor.py        # Redis consumer → aggregation → anomaly detection
├── alerts/
│   ├── engine.py           # YAML rule engine with cooldowns
│   ├── slack.py            # Slack Block Kit notifications
│   └── pagerduty.py        # PagerDuty Events API v2
└── dashboards/
    ├── grafana/             # Pre-built Grafana dashboard JSON
    └── streamlit_app.py     # Analytics dashboard with call replay
```

## Development

```bash
pip install -e ".[dev]"
pytest
ruff check .
mypy voicemon/
```

## License

Apache-2.0
