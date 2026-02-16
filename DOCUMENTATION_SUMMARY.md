# VoiceMon Documentation Summary

## 📚 Documentation Overview

This repository contains comprehensive architecture and system design documentation for **VoiceMon** - a production-grade observability platform for Voice AI agents.

### Available Documents

1. **[ARCHITECTURE.md](ARCHITECTURE.md)** (955 lines)
   - Complete High-Level Design (HLD)
   - Detailed Low-Level Design (LLD)
   - Component-by-component breakdown
   - Best for: Developers implementing or extending VoiceMon

2. **[SYSTEM_DESIGN.md](SYSTEM_DESIGN.md)** (863 lines)
   - System context and problem statement
   - Data flow diagrams
   - Scalability and performance analysis
   - Deployment patterns
   - Best for: Architects, DevOps, and infrastructure teams

3. **[README.md](README.md)**
   - Quick start guide
   - Installation instructions
   - Usage examples
   - Best for: Getting started quickly

---

## 🎯 Quick Architecture Summary

### What is VoiceMon?

VoiceMon is an observability platform that monitors Voice AI agents across 4 layers:

```
Layer 4: OUTCOME (Business)    → Task completion, CSAT, cost
Layer 3: UX (Perceived)        → E2E latency, interruptions
Layer 2: EXECUTION (Pipeline)  → STT/LLM/TTS latencies
Layer 1: INFRASTRUCTURE        → Jitter, packet loss, MOS score
```

### Core Architecture

```
Voice Agent (LiveKit/Pipecat/Vapi)
        ↓ (1-line SDK integration)
VoiceMonCollector (Python SDK)
        ↓ (Export telemetry)
Redis Streams (Event ingestion)
        ↓ (Consumer groups)
Metrics Processor Worker (Aggregation + Alerts)
        ↓ (Persist)
TimescaleDB (Time-series + Relational storage)
        ↓ (Query)
Grafana + Streamlit (Dashboards)
```

### Key Components

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **SDK** | Python (Pydantic) | Collect voice telemetry from agents |
| **Ingestion** | Redis Streams | Lightweight event pipeline |
| **Processing** | Python (AsyncIO) | Aggregate metrics, detect anomalies, fire alerts |
| **Storage** | TimescaleDB (PostgreSQL) | Time-series + relational queries |
| **Metrics** | Prometheus | SLO tracking, histogram quantiles |
| **Ops Dashboard** | Grafana | Real-time operational monitoring |
| **Analytics** | Streamlit | Call replay, drift detection, custom queries |
| **Alerting** | Slack + PagerDuty | Incident response and ChatOps |

---

## 🏗️ High-Level Design (HLD)

### System Layers

1. **Collection Layer** (Client-side SDK)
   - Integrations: LiveKit, Pipecat, Vapi
   - Models: Sessions, Turns, Events (4-layer framework)
   - Exporters: Console, Redis, Prometheus, OpenTelemetry

2. **Ingestion Layer** (Redis Streams)
   - Consumer groups for horizontal scaling
   - At-least-once delivery guarantee
   - Bounded memory (100k messages per stream)

3. **Processing Layer** (Worker Service)
   - Async consumer (xreadgroup)
   - Rolling window aggregation (100 samples)
   - Z-score anomaly detection (3σ threshold)
   - Alert evaluation with cooldown management

4. **Storage Layer** (TimescaleDB)
   - Hypertables for time-series events
   - Regular tables for sessions/turns
   - Continuous aggregates (hourly rollups)
   - Data retention policies (30 days raw, 365 days aggregates)

5. **Alerting Layer**
   - YAML-based alert rules
   - Severity routing (Slack for all, PagerDuty for critical)
   - Cooldown mechanism (prevent alert storms)

6. **Visualization Layer**
   - Grafana: Operational dashboards (Prometheus + TimescaleDB)
   - Streamlit: Analytical dashboards (TimescaleDB only)

### Data Flow

```
Agent Event → SDK → Redis Stream → Worker → TimescaleDB → Dashboard
                                      ↓
                                  Alert Engine → Slack/PagerDuty
```

---

## 🔬 Low-Level Design (LLD)

### Core Data Models

**VoiceSession** (Parent entity)
- Tracks: agent_id, framework, duration, status, cost
- Relationships: 1 session → N turns → N events

**Turn** (Conversation exchange)
- Tracks: speaker (user/agent), turn_number, duration, interrupted
- Contains: STT → LLM → TTS → UX events for that turn

**Events** (4-Layer Framework)
- Layer 1: `InfraEvent` (jitter, packet loss, MOS)
- Layer 2: `STTEvent`, `LLMEvent`, `TTSEvent` (pipeline metrics)
- Layer 3: `UXEvent` (e2e latency, interruptions)
- Layer 4: `OutcomeEvent` (task success, escalation)

### Key Algorithms

**1. Anomaly Detection (Z-Score)**
```python
z_score = (current_value - mean) / stdev
if abs(z_score) > 3.0:
    fire_alert("anomaly detected")
```

**2. Alert Cooldown**
```python
if time.now() - last_fire_time < cooldown_minutes * 60:
    skip_alert()  # Prevent spam
```

**3. Rolling Window Aggregation**
```python
_windows["stt_latency"] = deque(maxlen=100)
_windows["stt_latency"].append(latency_ms)  # O(1) operation
```

---

## 📊 System Design Patterns

### 1. Multi-Exporter Pattern
- **Problem**: Different consumers need different formats
- **Solution**: Plugin-based exporters (Console, Redis, Prometheus)
- **Benefit**: Add destinations without modifying collector

### 2. Consumer Group Pattern
- **Problem**: Single worker can't handle high volume
- **Solution**: Redis consumer groups for horizontal scaling
- **Benefit**: Load sharing, at-least-once delivery

### 3. Hypertable Partitioning
- **Problem**: Time-series queries slow on large tables
- **Solution**: TimescaleDB automatic partitioning by time
- **Benefit**: Recent data fast, old data auto-evicted

### 4. Anomaly Detection (Z-Score)
- **Problem**: Static thresholds miss context
- **Solution**: Statistical outlier detection
- **Benefit**: Adaptive, no ML overhead

---

## 🚀 Deployment Architecture

### Development (Docker Compose)

```bash
docker compose up -d
# Starts: TimescaleDB, Redis, Prometheus, Grafana, Worker, Streamlit
```

**Resource Usage**: 8-core, 16GB RAM, 500GB SSD

### Production (Kubernetes)

```
Kubernetes Cluster
├─ TimescaleDB StatefulSet (1 primary + 2 read replicas)
├─ Redis StatefulSet (1 primary + 2 replicas, Sentinel)
├─ Worker Deployment (HPA: 3-10 pods)
├─ Grafana Deployment (2 pods, load balanced)
└─ Streamlit Deployment (1 pod, sticky sessions)
```

**Scaling Strategy**:
- Workers: Horizontal (HPA on queue depth)
- Database: Vertical + read replicas
- Redis: Vertical (memory-bound)

---

## 📈 Performance Characteristics

### Throughput
- **Single Worker**: ~1,000 events/sec
- **Worker Pod (3 replicas)**: ~3,000 events/sec
- **Redis Streams**: ~50,000 events/sec
- **TimescaleDB**: ~10,000 INSERTs/sec

### Latency
- **SDK Overhead**: <1ms (in-memory)
- **Redis Export**: <5ms P95
- **End-to-End (Event → Alert)**: <2s P95
- **Dashboard Query**: <100ms P95

### Storage
- **Raw Events**: ~2GB/day (1M events)
- **Compressed**: ~0.6GB/day (70% reduction)
- **30-Day Retention**: ~5GB compressed

---

## 🔐 Security Considerations

1. **Authentication**: OAuth2/SAML for Grafana, BasicAuth for Streamlit
2. **Data Privacy**: PII in transcripts, column-level encryption if needed
3. **Network**: TLS for all connections, internal services not exposed
4. **Secrets**: Kubernetes Secrets for Slack/PagerDuty keys

---

## 🛠️ Technology Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| SDK | Python 3.10+, Pydantic v2 | Voice AI ecosystem is Python-first |
| Ingestion | Redis Streams | Lightweight, consumer groups, at-least-once |
| Processing | Python AsyncIO | Native async support, simple deployment |
| Storage | TimescaleDB (PostgreSQL) | Relational + time-series, SQL queries |
| Metrics | Prometheus | Industry standard, histogram quantiles |
| Dashboards | Grafana + Streamlit | Ops (Grafana) + analytics (Streamlit) |
| Alerts | Slack + PagerDuty | ChatOps + incident management |

---

## 📖 Document Navigation

### For Developers
1. Start with [README.md](README.md) for quick start
2. Read [ARCHITECTURE.md](ARCHITECTURE.md) for component details
3. Check code in `voicemon/core/` for implementation

### For Architects
1. Start with [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md) for system context
2. Review data flow diagrams
3. Check deployment patterns section

### For DevOps
1. Review [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md) deployment section
2. Check `docker-compose.yml` for stack setup
3. Review scalability and performance sections

### For Product/Business
1. Start with System Overview in [ARCHITECTURE.md](ARCHITECTURE.md)
2. Review 4-Layer Framework explanation
3. Check outcome metrics and cost tracking

---

## 🎓 Key Concepts Explained

### 4-Layer Voice Observability Framework

Inspired by [Hamming AI's framework](https://www.hamming.ai/blog/voice-agent-observability):

**Layer 1: Infrastructure**
- **What**: Network quality, codec performance
- **Why**: Bad network → poor audio quality → bad STT
- **Metrics**: Jitter, packet loss, MOS score, bitrate

**Layer 2: Execution**
- **What**: STT/LLM/TTS pipeline performance
- **Why**: Direct control, optimization opportunities
- **Metrics**: Latencies (STT, LLM TTFT, TTS TTFB), confidence scores, token usage

**Layer 3: User Experience**
- **What**: Perceived conversation quality
- **Why**: User doesn't care about STT latency, cares about response time
- **Metrics**: E2E latency, interruptions, silence ratio, TTFW

**Layer 4: Outcome**
- **What**: Business impact
- **Why**: Technical metrics don't matter if business goals not met
- **Metrics**: Task completion, CSAT, escalation rate, cost per call

### Consumer Groups (Redis Streams)

**Problem**: Single consumer can't handle 10k events/sec

**Solution**: Multiple workers, each processes subset of messages

```
Redis Stream: [msg1, msg2, msg3, msg4, msg5, msg6, msg7, msg8]
                ↓      ↓      ↓      ↓      ↓      ↓      ↓      ↓
Consumer Group: voicemon-workers
                ↓             ↓             ↓             ↓
             worker-1      worker-2      worker-3      worker-4
```

**Benefits**:
- Horizontal scaling (add workers without coordination)
- Load balancing (Redis distributes messages)
- At-least-once delivery (message not deleted until ACKed)

### Z-Score Anomaly Detection

**Problem**: Static threshold (e.g., STT latency > 500ms) doesn't account for context

**Solution**: Detect statistical outliers

```
Normal distribution: 99.7% of values within 3 standard deviations (3σ)
If current value is >3σ away from mean → anomaly
```

**Example**:
```
Last 50 STT latencies: [100, 105, 98, 102, 99, ..., 450]
Mean: 101ms
Stdev: 10ms
Current: 450ms
Z-score: (450 - 101) / 10 = 34.9 🚨 ANOMALY!
```

### Hypertables (TimescaleDB)

**Problem**: Time-series table with billions of rows → slow queries

**Solution**: Automatic partitioning by time (chunks)

```
stt_events table
├─ Chunk 1: Jan 1-7 (7 days of data)
├─ Chunk 2: Jan 8-14
├─ Chunk 3: Jan 15-21
└─ Chunk 4: Jan 22-28

Query: SELECT * FROM stt_events WHERE time > '2024-01-20'
→ Only scans Chunks 3-4 (fast!)
```

**Benefits**:
- Recent data fast (only scan latest chunks)
- Old data auto-deleted (retention policy)
- Compression (70% reduction after 1 day)

---

## 🔍 Common Queries

### How do I add a new metric?

1. Add field to event model (`voicemon/core/models.py`)
   ```python
   class STTEvent(BaseModel):
       my_new_metric: float | None = None
   ```

2. Update collector method (`voicemon/core/collector.py`)
   ```python
   async def record_stt(self, ..., my_new_metric: float | None = None):
       event = STTEvent(..., my_new_metric=my_new_metric)
   ```

3. Update TimescaleDB schema (`voicemon/storage/schema.sql`)
   ```sql
   ALTER TABLE stt_events ADD COLUMN my_new_metric DOUBLE PRECISION;
   ```

4. Update worker aggregation (`voicemon/workers/processor.py`)
   ```python
   self._windows["my_new_metric"].append(data["my_new_metric"])
   ```

### How do I add a new alert rule?

Edit `alert_rules.yaml`:
```yaml
rules:
  - name: my_custom_alert
    metric: my_new_metric
    operator: ">"
    threshold: 100
    severity: warning
    cooldown_minutes: 5
    notify: [slack]
```

Worker automatically reloads rules on restart.

### How do I scale to 10x traffic?

1. **Horizontal worker scaling**:
   ```bash
   kubectl scale deployment voicemon-worker --replicas=10
   ```

2. **Database read replicas**:
   - Add 2-3 read replicas for dashboard queries
   - Point Grafana/Streamlit to replicas

3. **Redis monitoring**:
   - Check queue depth: `redis-cli XLEN voicemon:stt`
   - If >1000, add more workers

---

## 📊 Metrics Glossary

| Metric | Description | Good | Warning | Critical |
|--------|-------------|------|---------|----------|
| **E2E Latency** | User speech → Agent first word | <800ms | <1200ms | >1800ms |
| **STT Latency** | Audio → Transcript | <200ms | <350ms | >1000ms |
| **LLM TTFT** | Request → First token | <600ms | <1000ms | >2000ms |
| **TTS TTFB** | Text → First audio byte | <150ms | <250ms | >800ms |
| **STT Confidence** | Transcription accuracy | >0.90 | >0.85 | <0.70 |
| **Jitter** | Network timing variance | <30ms | <50ms | >100ms |
| **Packet Loss** | Network packet drops | <0.5% | <1% | >3% |
| **MOS Score** | Mean Opinion Score (audio quality) | >4.0 | >3.5 | <3.0 |
| **Task Success** | Conversation objective achieved | >80% | >60% | <50% |

---

## 🤝 Contributing

See individual documents for specific areas:
- Code changes: [ARCHITECTURE.md](ARCHITECTURE.md) → Component Design
- Infrastructure: [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md) → Deployment Patterns
- New integrations: [README.md](README.md) → Integration Examples

---

## 📝 License

Apache-2.0 - see [LICENSE](LICENSE)

---

## 🙋 Support

- **Questions**: Open a GitHub issue
- **Bugs**: Include logs, config, and reproduction steps
- **Features**: Describe use case and expected behavior

---

**Last Updated**: 2024-02-16

**Documentation Version**: 1.0.0

**VoiceMon Version**: 0.1.0
