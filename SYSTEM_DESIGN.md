# VoiceMon System Design Document

## Executive Summary

VoiceMon is a full-stack observability platform designed specifically for Voice AI applications. This document provides a comprehensive system design overview covering architecture, data flows, scalability considerations, and operational patterns.

---

## Table of Contents

1. [System Context](#system-context)
2. [Architecture Layers](#architecture-layers)
3. [Component Design](#component-design)
4. [Data Flow Diagrams](#data-flow-diagrams)
5. [Scalability & Performance](#scalability--performance)
6. [Reliability & Fault Tolerance](#reliability--fault-tolerance)
7. [Security Architecture](#security-architecture)
8. [Deployment Patterns](#deployment-patterns)

---

## System Context

### Problem Statement

Voice AI agents (built on LiveKit, Pipecat, or Vapi) require specialized observability that traditional APM tools don't provide:

1. **Voice-Specific Metrics**: STT confidence, TTS audio quality, turn-taking dynamics
2. **Multi-Layer Visibility**: Infrastructure (jitter) → Execution (latency) → UX (perceived quality) → Outcome (task success)
3. **Real-Time Alerting**: Sub-second latency spikes cause conversation breakdown
4. **Cost Attribution**: Per-session token usage, API costs, ROI tracking

### Solution Overview

VoiceMon implements a **4-Layer Voice Observability Framework**:

```
┌───────────────────────────────────────────────────────────────┐
│ Layer 4: OUTCOME (Business Impact)                           │
│ • Task completion rate, CSAT, escalation rate, cost per call │
├───────────────────────────────────────────────────────────────┤
│ Layer 3: USER EXPERIENCE (Perceived Quality)                 │
│ • E2E latency, interruptions, silence ratio, TTFW            │
├───────────────────────────────────────────────────────────────┤
│ Layer 2: EXECUTION (Pipeline Performance)                    │
│ • STT latency/confidence, LLM TTFT/tokens, TTS TTFB         │
├───────────────────────────────────────────────────────────────┤
│ Layer 1: INFRASTRUCTURE (Network & Codec)                    │
│ • Jitter, packet loss, MOS score, bitrate, RTT              │
└───────────────────────────────────────────────────────────────┘
```

### Stakeholders

| Role | Needs | VoiceMon Feature |
|------|-------|------------------|
| Voice Engineers | Pipeline debugging, latency waterfalls | Streamlit call replay, turn-level traces |
| SREs / DevOps | Uptime, SLA monitoring, incident response | Grafana dashboards, PagerDuty integration |
| Product Managers | Task success rates, user experience metrics | Outcome analytics, CSAT trending |
| Finance / Ops | Cost per conversation, ROI tracking | Token usage reports, cost attribution |

---

## Architecture Layers

### L1: Client Layer (Voice AI Agents)

**Components**:
- LiveKit Agents (Python SDK)
- Pipecat Pipelines (Python SDK)
- Vapi Applications (Webhooks + REST API)

**Integration Point**:
```python
# 1-line instrumentation
from voicemon.integrations.livekit import instrument_livekit
session_id = instrument_livekit(agent_session, collector, agent_id="my-agent")
```

**Design Principle**: **Non-invasive** — VoiceMon hooks into existing framework events, requires no agent code modification.

---

### L2: Collection Layer (SDK)

**Responsibility**: Capture, validate, and export voice telemetry events.

```
┌────────────────────────────────────────────────────────────┐
│                    VoiceMonCollector                       │
│                                                            │
│  ┌──────────────────┐   ┌──────────────────┐             │
│  │ Session Manager  │   │  Turn Manager    │             │
│  │ • start_session  │   │  • start_turn    │             │
│  │ • end_session    │   │  • end_turn      │             │
│  └──────────────────┘   └──────────────────┘             │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │         Event Recorders (4-Layer Framework)          │ │
│  │  • record_infra()  — Layer 1                         │ │
│  │  • record_stt()    — Layer 2                         │ │
│  │  • record_llm()    — Layer 2                         │ │
│  │  • record_tts()    — Layer 2                         │ │
│  │  • record_ux()     — Layer 3                         │ │
│  │  • record_outcome()— Layer 4                         │ │
│  └──────────────────────────────────────────────────────┘ │
│                          │                                │
│                          ▼                                │
│  ┌──────────────────────────────────────────────────────┐ │
│  │           Exporter Plugin System                     │ │
│  │  • ConsoleExporter    (stdout, debugging)           │ │
│  │  • RedisStreamsExporter (→ worker pipeline)         │ │
│  │  │  PrometheusExporter (→ /metrics endpoint)        │ │
│  │  • OpenTelemetryExporter (→ Jaeger/Tempo)           │ │
│  └──────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

**Key Design Decisions**:

1. **In-Memory State Management**
   - `_sessions: dict[str, VoiceSession]` — Active session cache
   - `_turns: dict[str, Turn]` — Active turn cache
   - `_turn_reports: dict[str, TurnReport]` — Aggregated turn data
   - **Why**: Fast lookups, aggregation before export
   - **Trade-off**: Limited by process memory, need periodic cleanup

2. **Async-First API**
   - All `record_*` methods are `async def`
   - **Why**: Non-blocking export to Redis/Prometheus
   - **Trade-off**: Requires async agent code (standard in voice AI)

3. **Multi-Exporter Pattern**
   - List of `BaseExporter` plugins
   - **Why**: Parallel export to multiple destinations (console + Redis + Prometheus)
   - **Trade-off**: Exporter failure doesn't block others (logged, non-fatal)

---

### L3: Ingestion Layer (Redis Streams)

**Responsibility**: Lightweight, at-least-once event ingestion with horizontal scaling.

```
┌──────────────────────────────────────────────────────────┐
│              Redis Streams (8 Event Types)               │
│                                                          │
│  voicemon:sessions   ━━━━━━━┓                           │
│  voicemon:turns      ━━━━━━━┫                           │
│  voicemon:infra      ━━━━━━━┫                           │
│  voicemon:stt        ━━━━━━━┫  Consumer Group:         │
│  voicemon:llm        ━━━━━━━┫  "voicemon-workers"      │
│  voicemon:tts        ━━━━━━━┫                           │
│  voicemon:ux         ━━━━━━━┫  ┌────────┐  ┌────────┐  │
│  voicemon:outcomes   ━━━━━━━╋━▶│worker-1│  │worker-2│  │
│                              ┃  └────────┘  └────────┘  │
│  Stream Properties:          ┃  ┌────────┐  ┌────────┐  │
│  • Max Length: 100k          ┗━▶│worker-3│  │worker-4│  │
│  • Eviction: Approximate        └────────┘  └────────┘  │
│  • Persistence: AOF                                      │
└──────────────────────────────────────────────────────────┘
```

**Why Redis Streams?**

| Feature | Benefit |
|---------|---------|
| **Consumer Groups** | Multiple workers share load, each message delivered to one consumer |
| **At-Least-Once Delivery** | XREADGROUP + XACK ensures no message loss |
| **Horizontal Scaling** | Add workers without coordination, Redis load balances |
| **Bounded Memory** | MAXLEN with approximate trimming (LRU eviction) |
| **Low Latency** | <5ms P95 write latency (local network) |
| **Simple Operations** | No Kafka/ZooKeeper complexity, single Redis instance sufficient |

**Stream Schema**:
```json
{
  "event_type": "stt",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "session_id": "abc123",
    "turn_id": "def456",
    "transcript": "hello world",
    "confidence": 0.95,
    "latency_ms": 120,
    "provider": "deepgram",
    "model": "nova-2"
  }
}
```

---

### L4: Processing Layer (Worker)

**Responsibility**: Consume events, aggregate metrics, detect anomalies, fire alerts, persist to database.

```
┌───────────────────────────────────────────────────────────┐
│               MetricsProcessor (Worker Pod)               │
│                                                           │
│  ┌────────────────────────────────────────────────────┐   │
│  │  Consumer Loop (per event type)                   │   │
│  │                                                    │   │
│  │  while running:                                    │   │
│  │    messages = redis.xreadgroup(                    │   │
│  │        "voicemon-workers", "worker-1",            │   │
│  │        {"voicemon:stt": ">"}, count=100           │   │
│  │    )                                               │   │
│  │                                                    │   │
│  │    for msg_id, data in messages:                   │   │
│  │        ┌──────────────────────────────────────┐    │   │
│  │        │ 1. Persist to TimescaleDB           │    │   │
│  │        │    INSERT INTO stt_events (...)      │    │   │
│  │        └──────────────────────────────────────┘    │   │
│  │        ┌──────────────────────────────────────┐    │   │
│  │        │ 2. Update Rolling Windows            │    │   │
│  │        │    _windows["stt_latency"].append()  │    │   │
│  │        └──────────────────────────────────────┘    │   │
│  │        ┌──────────────────────────────────────┐    │   │
│  │        │ 3. Check Thresholds                  │    │   │
│  │        │    if latency > 1000: alert          │    │   │
│  │        └──────────────────────────────────────┘    │   │
│  │        ┌──────────────────────────────────────┐    │   │
│  │        │ 4. Detect Anomalies                  │    │   │
│  │        │    z_score = (x - μ) / σ             │    │   │
│  │        │    if |z| > 3.0: alert               │    │   │
│  │        └──────────────────────────────────────┘    │   │
│  │        ┌──────────────────────────────────────┐    │   │
│  │        │ 5. Fire Alerts (with cooldown)       │    │   │
│  │        │    AlertEngine.notify(alert)         │    │   │
│  │        └──────────────────────────────────────┘    │   │
│  │                                                    │   │
│  │    redis.xack("voicemon:stt", "voicemon-workers", │   │
│  │               *msg_ids)                            │   │
│  └────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────┘
```

**Anomaly Detection Algorithm**:

```python
def detect_anomaly(metric_window):
    """Z-score anomaly detection.
    
    Assumptions:
    - Metric follows normal distribution
    - 3-sigma rule: 99.7% of values within 3 std devs
    - Values beyond 3σ are statistical outliers
    """
    if len(metric_window) < 50:  # Need minimum samples
        return None
    
    values = list(metric_window)
    current = values[-1]
    mean = statistics.mean(values[:-1])
    stdev = statistics.stdev(values[:-1])
    
    if stdev == 0:  # No variance (constant metric)
        return None
    
    z_score = (current - mean) / stdev
    
    if abs(z_score) > 3.0:
        return Alert(
            severity="warning",
            message=f"Anomaly: {current:.1f}ms (μ={mean:.1f}, σ={stdev:.1f}, z={z_score:.1f})"
        )
```

**Why Z-Score?**
- ✅ Simple, no ML dependencies
- ✅ Adaptive to workload patterns (sliding window)
- ✅ Real-time detection (O(1) compute per event)
- ❌ Assumes normal distribution (not always true)
- ❌ Doesn't handle seasonality (use ML for that)

---

### L5: Storage Layer (TimescaleDB)

**Responsibility**: Durable, queryable storage for sessions, turns, and time-series events.

```
┌───────────────────────────────────────────────────────────┐
│                  TimescaleDB (PostgreSQL)                 │
│                                                           │
│  ┌────────────────────────────────────────────────────┐   │
│  │  Regular Tables (Relational)                       │   │
│  │                                                    │   │
│  │  sessions                                          │   │
│  │  ├─ id (PK, UUID)                                  │   │
│  │  ├─ agent_id (indexed)                            │   │
│  │  ├─ started_at (indexed DESC)                     │   │
│  │  ├─ status (indexed: active|completed|failed)     │   │
│  │  └─ metadata (JSONB)                              │   │
│  │                                                    │   │
│  │  turns                                             │   │
│  │  ├─ id (PK, UUID)                                  │   │
│  │  ├─ session_id (FK → sessions, cascade delete)    │   │
│  │  ├─ turn_number (indexed)                         │   │
│  │  ├─ speaker (user|agent)                          │   │
│  │  └─ started_at (indexed DESC)                     │   │
│  └────────────────────────────────────────────────────┘   │
│                                                           │
│  ┌────────────────────────────────────────────────────┐   │
│  │  Hypertables (Time-Series)                        │   │
│  │                                                    │   │
│  │  infra_events                                      │   │
│  │  ├─ time (partitioning key, indexed)              │   │
│  │  ├─ session_id (indexed)                          │   │
│  │  ├─ jitter_ms, packet_loss_pct, mos_score        │   │
│  │  └─ metadata (JSONB)                              │   │
│  │                                                    │   │
│  │  stt_events                                        │   │
│  │  ├─ time (partitioning key)                       │   │
│  │  ├─ session_id (indexed)                          │   │
│  │  ├─ latency_ms, confidence                        │   │
│  │  ├─ provider (indexed), model                     │   │
│  │  └─ transcript (TEXT)                             │   │
│  │                                                    │   │
│  │  llm_events, tts_events, ux_events (similar)      │   │
│  └────────────────────────────────────────────────────┘   │
│                                                           │
│  ┌────────────────────────────────────────────────────┐   │
│  │  Continuous Aggregates (Pre-computed Rollups)     │   │
│  │                                                    │   │
│  │  stt_hourly_stats                                  │   │
│  │  SELECT time_bucket('1 hour', time) AS bucket,    │   │
│  │         agent_id,                                  │   │
│  │         AVG(latency_ms),                          │   │
│  │         percentile_cont(0.95) WITHIN GROUP        │   │
│  │            (ORDER BY latency_ms) AS p95,          │   │
│  │         COUNT(*) AS event_count                   │   │
│  │  FROM stt_events                                   │   │
│  │  GROUP BY bucket, agent_id                        │   │
│  │                                                    │   │
│  │  Refresh Policy: Every 5 minutes                  │   │
│  │  Retention: 365 days                              │   │
│  └────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────┘
```

**Why TimescaleDB?**

| Requirement | TimescaleDB | InfluxDB | Prometheus |
|-------------|-------------|----------|------------|
| Relational JOINs (sessions→turns→events) | ✅ Native | ❌ Limited | ❌ No |
| Time-series partitioning | ✅ Automatic | ✅ Yes | ✅ Yes |
| SQL compatibility | ✅ PostgreSQL | ❌ InfluxQL | ❌ PromQL |
| Continuous aggregates | ✅ Materialized views | ✅ Downsampling | ❌ No |
| Data retention policies | ✅ Automatic | ✅ Yes | ✅ Yes |
| Text search (transcripts) | ✅ Full-text | ❌ Limited | ❌ No |
| Cost | 💰 Free (self-hosted) | 💰💰 Cloud pricing | 💰 Free |

**Data Retention Strategy**:

```sql
-- Raw hypertable events: 30 days
SELECT add_retention_policy('stt_events', INTERVAL '30 days');
SELECT add_retention_policy('llm_events', INTERVAL '30 days');
SELECT add_retention_policy('tts_events', INTERVAL '30 days');

-- Continuous aggregates (hourly): 365 days
SELECT add_retention_policy('stt_hourly_stats', INTERVAL '365 days');
SELECT add_retention_policy('llm_hourly_stats', INTERVAL '365 days');

-- Session/turn metadata: Forever (or manual cleanup)
-- These are lightweight, keep for compliance/audit
```

**Compression**:
```sql
-- Enable compression after 1 day
ALTER TABLE stt_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'session_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('stt_events', INTERVAL '1 day');

-- Result: ~70% storage reduction
```

---

### L6: Alerting Layer

**Responsibility**: Evaluate alert rules, route notifications, prevent alert storms.

```
┌───────────────────────────────────────────────────────────┐
│                      Alert Engine                         │
│                                                           │
│  ┌────────────────────────────────────────────────────┐   │
│  │  1. Load Rules (alert_rules.yaml)                 │   │
│  │                                                    │   │
│  │  rules:                                            │   │
│  │    - name: stt_latency_critical                   │   │
│  │      metric: stt_latency_ms                       │   │
│  │      operator: ">"                                │   │
│  │      threshold: 1000                              │   │
│  │      severity: critical                           │   │
│  │      cooldown_minutes: 3                          │   │
│  │      notify: [slack, pagerduty]                   │   │
│  └────────────────────────────────────────────────────┘   │
│                                                           │
│  ┌────────────────────────────────────────────────────┐   │
│  │  2. Evaluate Metric                               │   │
│  │                                                    │   │
│  │  if value > threshold:                            │   │
│  │      key = f"{rule_name}:{agent_id}"              │   │
│  │      if key in cooldowns and                      │   │
│  │         time.now() - cooldowns[key] < cooldown:   │   │
│  │          return  # Skip (still in cooldown)       │   │
│  │      else:                                         │   │
│  │          fire_alert()                             │   │
│  │          cooldowns[key] = time.now()              │   │
│  └────────────────────────────────────────────────────┘   │
│                                                           │
│  ┌────────────────────────────────────────────────────┐   │
│  │  3. Route Notification                            │   │
│  │                                                    │   │
│  │  if severity in ["info", "warning"]:              │   │
│  │      ┌─────────────────────────────────────┐      │   │
│  │      │ Slack Webhook                       │      │   │
│  │      │ POST https://hooks.slack.com/...    │      │   │
│  │      │ Body: Block Kit formatted message   │      │   │
│  │      └─────────────────────────────────────┘      │   │
│  │                                                    │   │
│  │  if severity in ["critical", "p0"]:               │   │
│  │      ┌─────────────────────────────────────┐      │   │
│  │      │ Slack + @oncall mention             │      │   │
│  │      └─────────────────────────────────────┘      │   │
│  │      ┌─────────────────────────────────────┐      │   │
│  │      │ PagerDuty Events API v2             │      │   │
│  │      │ POST https://events.pagerduty.com   │      │   │
│  │      │ Body: {                             │      │   │
│  │      │   "routing_key": "...",             │      │   │
│  │      │   "event_action": "trigger",        │      │   │
│  │      │   "payload": {...}                  │      │   │
│  │      │ }                                    │      │   │
│  │      └─────────────────────────────────────┘      │   │
│  └────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────┘
```

**Alert Routing Matrix**:

| Severity | Slack Channel | PagerDuty | Oncall Mention | Use Case |
|----------|---------------|-----------|----------------|----------|
| `info` | #voice-metrics | ❌ | ❌ | FYI, trends, capacity planning |
| `warning` | #voice-alerts | ❌ | ❌ | Approaching threshold, investigate |
| `critical` | #voice-oncall | ✅ | ✅ | SLA breach imminent, action required |
| `p0` | #voice-oncall | ✅ | ✅ | Production outage, page immediately |

**Cooldown Mechanism**:

```python
_cooldowns: dict[str, float] = {}  # {rule_key: last_fire_timestamp}

def should_fire_alert(rule_name, agent_id, cooldown_minutes):
    key = f"{rule_name}:{agent_id}"
    now = time.monotonic()
    
    if key in _cooldowns:
        elapsed = now - _cooldowns[key]
        if elapsed < cooldown_minutes * 60:
            return False  # Still in cooldown
    
    _cooldowns[key] = now
    return True
```

**Why Cooldowns?**
- Prevent alert storms (same alert firing every 5 seconds)
- Reduce noise, maintain signal-to-noise ratio
- Give time for incident responders to acknowledge/resolve
- Default: 5 minutes (configurable per rule)

---

### L7: Visualization Layer

```
┌─────────────────────────────────────────────────────────┐
│                   Grafana (Ops Dashboard)               │
│                                                         │
│  Datasources:                                           │
│  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │ Prometheus      │  │ TimescaleDB                 │  │
│  │ • Histograms    │  │ • Session/Turn metadata     │  │
│  │ • Counters      │  │ • Time-series events        │  │
│  │ • Gauges        │  │ • SQL queries               │  │
│  └─────────────────┘  └─────────────────────────────┘  │
│                                                         │
│  Dashboards:                                            │
│  • Voice Agent Overview (latency, throughput, errors)  │
│  • STT Performance (latency, confidence, provider)     │
│  • LLM Performance (TTFT, tokens/sec, cost)            │
│  • TTS Performance (TTFB, audio quality)               │
│  • Infrastructure (jitter, packet loss, MOS)           │
│  • Alerting (active alerts, history)                   │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                Streamlit (Analytics Dashboard)          │
│                                                         │
│  Features:                                              │
│  • Call Replay: Step through turns, view metrics       │
│  • Session Search: Filter by agent, date, status       │
│  • Drift Detection: STT confidence degradation         │
│  • Cost Analytics: Token usage, API costs, trends      │
│  • Custom Queries: SQL editor + result visualization   │
│                                                         │
│  Datasource: TimescaleDB (PostgreSQL)                  │
└─────────────────────────────────────────────────────────┘
```

---

## Data Flow Diagrams

### End-to-End Telemetry Flow

```
┌────────────────────────────────────────────────────────────────────┐
│ 1. Voice Agent (LiveKit)                                           │
│    User says "hello" → STT transcribes → LLM responds → TTS speaks │
└───────────────────────────┬────────────────────────────────────────┘
                            │ Framework events
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│ 2. VoiceMon SDK (Integration Layer)                               │
│    • instrument_livekit() hooks into agent_session events         │
│    • Extracts metrics: transcript, confidence, latency, tokens    │
│    • Calls collector.record_stt(...)                              │
└───────────────────────────┬────────────────────────────────────────┘
                            │ Event objects
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│ 3. VoiceMonCollector                                              │
│    • Validates event with Pydantic schema                         │
│    • Updates turn_reports (aggregate per turn)                    │
│    • Emits to all exporters in parallel                           │
└──────────┬─────────────────┬──────────────────┬────────────────────┘
           │                 │                  │
           ▼                 ▼                  ▼
┌──────────────────┐  ┌──────────────┐  ┌─────────────────┐
│ ConsoleExporter  │  │ RedisExporter│  │PrometheusExporter│
│ (stdout logging) │  │(XADD stream) │  │(histogram.observe)│
└──────────────────┘  └──────┬───────┘  └─────────────────┘
                             │
                             ▼
┌────────────────────────────────────────────────────────────────────┐
│ 4. Redis Streams                                                  │
│    • Stream: voicemon:stt                                         │
│    • Message: {event_type, timestamp, data}                       │
│    • Consumer Group: voicemon-workers                             │
└───────────────────────────┬────────────────────────────────────────┘
                            │
                            ▼ XREADGROUP (blocking, batch=100)
┌────────────────────────────────────────────────────────────────────┐
│ 5. MetricsProcessor Worker                                        │
│    • Persist to TimescaleDB: INSERT INTO stt_events (...)         │
│    • Update rolling windows: deque.append(latency_ms)             │
│    • Check thresholds: if latency > 1000ms → alert                │
│    • Detect anomalies: z_score = (x - μ) / σ                      │
│    • Fire alerts: AlertEngine.notify(...)                         │
│    • ACK message: XACK voicemon:stt voicemon-workers msg_id       │
└─────────┬──────────────────┬───────────────────┬──────────────────┘
          │                  │                   │
          ▼                  ▼                   ▼
┌─────────────────┐  ┌───────────────┐  ┌──────────────────┐
│ TimescaleDB     │  │ Alert Engine  │  │ Rolling Stats    │
│ • stt_events    │  │ • Slack       │  │ • Mean, P95, P99 │
│ • stt_hourly_*  │  │ • PagerDuty   │  │ • Dashboard API  │
└────────┬────────┘  └───────────────┘  └──────────────────┘
         │
         ▼ SQL queries
┌────────────────────────────────────────────────────────────────────┐
│ 6. Visualization Layer                                            │
│    • Grafana: SELECT percentile_cont(0.95) ... FROM stt_events    │
│    • Streamlit: SELECT * FROM sessions WHERE agent_id = ?         │
└────────────────────────────────────────────────────────────────────┘
```

---

## Scalability & Performance

### Horizontal Scaling Points

```
┌────────────────────────────────────────────────────────┐
│ Component              │ Scaling Strategy              │
├────────────────────────┼───────────────────────────────┤
│ VoiceMonCollector      │ N:1 (per voice agent process) │
│ (embedded in agent)    │ No coordination needed        │
├────────────────────────┼───────────────────────────────┤
│ Redis Streams          │ Vertical (single instance)    │
│                        │ → Redis Cluster (sharding)    │
├────────────────────────┼───────────────────────────────┤
│ MetricsProcessor       │ Horizontal (consumer groups)  │
│ (Worker Pods)          │ • Add pods: kubectl scale     │
│                        │ • Unique consumer names       │
│                        │ • Redis load balances         │
├────────────────────────┼───────────────────────────────┤
│ TimescaleDB            │ Vertical (bigger instance)    │
│                        │ + Read replicas (for queries) │
│                        │ + Connection pooling (PgBouncer)
├────────────────────────┼───────────────────────────────┤
│ Grafana                │ Horizontal (stateless)        │
│                        │ • Load balancer in front      │
├────────────────────────┼───────────────────────────────┤
│ Streamlit              │ Limited (stateful sessions)   │
│                        │ • Sticky sessions required    │
└────────────────────────┴───────────────────────────────┘
```

### Performance Targets

| Metric | Target | Strategy |
|--------|--------|----------|
| **Event Ingestion** | 10k events/sec | Redis Streams, batch writes |
| **End-to-End Latency** | <2s (P95) | Async export, non-blocking workers |
| **Worker Processing** | 1k events/sec per pod | Horizontal scaling (3-5 pods) |
| **Database Writes** | 5k INSERTs/sec | TimescaleDB hypertables, batch commits |
| **Dashboard Queries** | <200ms (P95) | Continuous aggregates, read replicas |
| **Storage Growth** | 2GB/day (1M events) | Compression (70% reduction), retention policies |

### Bottleneck Analysis

**Scenario: 100 concurrent voice agents, 1M events/day**

```
Voice Agents (100)
  │
  ├─ Each agent: ~10 events/session × 100 sessions/day = 1k events/day
  ├─ Total: 100k events/day (~1 event/sec sustained, 10 events/sec peak)
  │
  ▼
Redis Streams
  │ Capacity: 50k events/sec (write)
  │ Headroom: 5000x
  │ Bottleneck: ❌ No
  │
  ▼
MetricsProcessor (3 pods)
  │ Capacity: 3k events/sec (3 pods × 1k events/sec)
  │ Headroom: 300x
  │ Bottleneck: ❌ No
  │
  ▼
TimescaleDB (8-core instance)
  │ Capacity: 5k INSERTs/sec
  │ Headroom: 500x
  │ Bottleneck: ❌ No
  │
  ▼
Grafana Queries (hourly aggregates)
  │ Query time: <100ms (P95)
  │ Bottleneck: ❌ No
```

**Conclusion**: Current architecture handles 100 agents with 50x headroom. Scale beyond 5k agents requires:
- Redis Cluster (sharding)
- TimescaleDB replication (read replicas for dashboards)
- Worker auto-scaling (HPA on queue depth)

---

## Reliability & Fault Tolerance

### Failure Modes & Mitigation

| Component | Failure Mode | Impact | Mitigation |
|-----------|--------------|--------|-----------|
| **VoiceMonCollector** | Agent crash | ❌ In-flight events lost | Non-critical; next session starts fresh |
| **Redis Streams** | Redis OOM | 🔴 Event ingestion stops | MAXLEN trimming, LRU eviction, monitoring |
| **Redis Streams** | Redis crash | 🟡 Events buffered in SDK | AOF persistence, Sentinel for HA |
| **Worker Pod** | Crash mid-processing | 🟡 Messages redelivered | Consumer groups (at-least-once), idempotent writes |
| **TimescaleDB** | Primary failure | 🔴 No writes | HA setup (Patroni), read replicas promoted |
| **Grafana** | Pod crash | 🟢 Dashboard unavailable | Stateless, auto-restart, load balancer |
| **Slack Webhook** | Rate limit / timeout | 🟡 Alert not delivered | Retry with exponential backoff, fallback to logs |
| **PagerDuty** | API down | 🟡 Incident not created | Retry, fallback to Slack, manual escalation |

### Data Durability Guarantees

```
┌────────────────────────────────────────────────────────┐
│ Event Journey             │ Durability                 │
├───────────────────────────┼────────────────────────────┤
│ 1. Agent → Collector      │ ❌ No (in-memory)          │
│ 2. Collector → Redis      │ ✅ Yes (AOF on disk)       │
│ 3. Redis → Worker         │ ✅ Yes (pending ACK)       │
│ 4. Worker → TimescaleDB   │ ✅ Yes (WAL, replication)  │
└───────────────────────────┴────────────────────────────┘
```

**SLA**: Once event reaches Redis Streams, durability guaranteed (at-least-once delivery).

---

## Security Architecture

### Threat Model

| Threat | Mitigation |
|--------|-----------|
| **Unauthorized access to dashboards** | OAuth2/SAML for Grafana, BasicAuth for Streamlit |
| **PII in transcripts** | Column-level encryption, data masking, retention policies |
| **Redis command injection** | No user-controlled Redis commands, parameterized queries only |
| **SQL injection** | Parameterized queries (`$1, $2`), no string concatenation |
| **Slack webhook leaked** | Environment variable, Kubernetes Secret, rotate quarterly |
| **PagerDuty key leaked** | Environment variable, Kubernetes Secret, rotate quarterly |
| **Man-in-the-middle** | TLS for all network connections (Redis, TimescaleDB, webhooks) |

### Network Security (Kubernetes)

```
┌────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                    │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  External Ingress (TLS termination)              │  │
│  │  • grafana.voicemon.example.com                  │  │
│  │  • streamlit.voicemon.example.com                │  │
│  └──────────────────┬───────────────────────────────┘  │
│                     │ (HTTPS)                          │
│                     ▼                                  │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Internal Network (ClusterIP)                    │  │
│  │  • voicemon-grafana:3000                         │  │
│  │  • voicemon-streamlit:8501                       │  │
│  │  • voicemon-timescale:5432 (no external access)  │  │
│  │  • voicemon-redis:6379 (no external access)      │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  NetworkPolicy:                                        │
│  • Worker → TimescaleDB (allow)                       │
│  • Worker → Redis (allow)                             │
│  • Grafana → TimescaleDB (allow, read-only)           │
│  • Streamlit → TimescaleDB (allow)                    │
│  • External → TimescaleDB (deny)                      │
│  • External → Redis (deny)                            │
└────────────────────────────────────────────────────────┘
```

---

## Deployment Patterns

### Pattern 1: Single-Tenant (Self-Hosted)

**Use Case**: Small team, <100 agents, self-managed infrastructure

```
Docker Compose (single host)
├─ TimescaleDB (single instance)
├─ Redis (single instance)
├─ Prometheus (single instance)
├─ Grafana (single instance)
├─ Worker (single process)
└─ Streamlit (single process)

Resources: 8-core, 16GB RAM, 500GB SSD
Cost: ~$100/month (AWS t3.xlarge + EBS)
```

### Pattern 2: Multi-Tenant (Managed Service)

**Use Case**: SaaS offering, 1000s of agents, multi-customer isolation

```
Kubernetes (GKE/EKS)
├─ TimescaleDB (per-tenant database, shared instance)
├─ Redis (shared, namespace isolation)
├─ Worker (shared, HPA: 5-20 pods)
├─ Grafana (shared, per-tenant dashboards)
└─ Streamlit (per-tenant pods)

Tenant Isolation:
• Database: Row-level security (tenant_id column)
• Redis: Key prefix (tenant:{id}:*)
• Dashboards: Grafana organizations
• API: JWT authentication with tenant claim
```

### Pattern 3: Edge Deployment (Low Latency)

**Use Case**: Voice agents in multiple regions, <100ms export latency

```
Multi-Region Deployment
├─ us-east-1
│  ├─ Worker (processes regional events)
│  ├─ Redis (regional stream)
│  └─ TimescaleDB (regional replica)
├─ eu-west-1
│  ├─ Worker (processes regional events)
│  ├─ Redis (regional stream)
│  └─ TimescaleDB (regional replica)
└─ Global
   ├─ TimescaleDB (primary, aggregates from regions)
   ├─ Grafana (global dashboards)
   └─ Streamlit (global analytics)

Data Flow:
Agent (us-east-1) → Redis (us-east-1) → Worker (us-east-1) → TimescaleDB (us-east-1)
                                                              ↓ (async replication)
                                                              TimescaleDB (global)
```

---

## Operational Metrics

### SLIs (Service Level Indicators)

| SLI | Target | Measurement |
|-----|--------|-------------|
| Event Ingestion Success Rate | 99.9% | `sum(redis_xadd_success) / sum(redis_xadd_total)` |
| Worker Processing Lag | <10s (P95) | `time.now() - event.timestamp` |
| Alert Delivery Success Rate | 99% | `sum(alert_delivered) / sum(alert_fired)` |
| Dashboard Query Latency | <500ms (P95) | Grafana query duration histogram |
| Data Loss Rate | <0.01% | Compare event count (SDK) vs (TimescaleDB) |

### Monitoring Dashboards

**Ops Dashboard (Grafana)**:
- Voice Agent Health: Active sessions, event rate, error rate
- Pipeline Latency: STT/LLM/TTS latency percentiles
- Infrastructure: Jitter, packet loss, MOS score
- Cost: Token usage, API costs, cost per session

**System Health Dashboard (Grafana)**:
- Redis: Stream depth, memory usage, eviction rate
- Worker: Processing rate, error rate, queue lag
- TimescaleDB: Query latency, replication lag, disk usage
- Alerts: Active alerts, alert rate, MTTR

---

## Conclusion

VoiceMon's architecture is designed for:

1. **Observability**: 4-layer framework captures infrastructure → outcome metrics
2. **Scalability**: Horizontal scaling (workers), vertical scaling (DB), sharding (Redis)
3. **Reliability**: At-least-once delivery, idempotent processing, HA components
4. **Performance**: <2s end-to-end latency, 10k events/sec throughput
5. **Simplicity**: Open-source stack, Docker Compose dev, Kubernetes prod

The system balances **pragmatism** (Redis Streams, Z-score anomalies) with **production-readiness** (TimescaleDB, PagerDuty integration), making it suitable for both startups and enterprises.

---

**Next Steps**:
- Review [ARCHITECTURE.md](ARCHITECTURE.md) for detailed component design
- See [README.md](README.md) for quick start guide
- Check [CONTRIBUTING.md](CONTRIBUTING.md) for development setup
