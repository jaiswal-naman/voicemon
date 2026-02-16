# VoiceMon Architecture Documentation

## Table of Contents
1. [System Overview](#system-overview)
2. [High-Level Design (HLD)](#high-level-design-hld)
3. [Low-Level Design (LLD)](#low-level-design-lld)
4. [System Design Patterns](#system-design-patterns)
5. [Data Flow](#data-flow)
6. [Deployment Architecture](#deployment-architecture)

---

## System Overview

**VoiceMon** is a production-grade observability platform for Voice AI agents, implementing the **4-Layer Voice Observability Framework**. It provides comprehensive monitoring, tracing, alerting, and analytics for voice agents built on LiveKit, Pipecat, or Vapi.

### Core Purpose
- **Monitor**: Real-time telemetry collection across all layers of voice AI pipeline
- **Analyze**: Statistical aggregation, anomaly detection, and trend analysis
- **Alert**: Threshold-based and anomaly-based alerting with smart routing
- **Visualize**: Operational dashboards (Grafana) and analytical insights (Streamlit)

### Key Metrics Tracked
1. **Layer 1 (Infrastructure)**: Jitter, packet loss, MOS score, bitrate, codec performance
2. **Layer 2 (Execution)**: STT/LLM/TTS latencies, confidence scores, token usage
3. **Layer 3 (User Experience)**: E2E latency, interruptions, silence ratio, TTFW
4. **Layer 4 (Outcome)**: Task completion, CSAT, escalation rate, cost per session

---

## High-Level Design (HLD)

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VOICE AI AGENTS (Client Layer)                   │
│              LiveKit Agents / Pipecat / Vapi Applications           │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 │ SDK Integration (1-line)
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                      COLLECTION LAYER                               │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │           VoiceMonCollector (Core SDK)                       │   │
│  │  • Session/Turn Management                                   │   │
│  │  • Event Recording (4-Layer Framework)                       │   │
│  │  • Multi-Exporter Pattern                                    │   │
│  └───────────┬──────────────────────────────┬───────────────────┘   │
│              │                              │                       │
│              │                              │                       │
│  ┌───────────▼────────────┐    ┌───────────▼──────────────┐        │
│  │  OpenTelemetry Bridge  │    │  Exporter Abstractions   │        │
│  │  (Distributed Tracing) │    │  • Console               │        │
│  └────────────────────────┘    │  • Redis Streams         │        │
│                                │  • Prometheus            │        │
│                                └──────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                  ┌──────────────┼──────────────┐
                  │              │              │
         ┌────────▼──────┐  ┌───▼────┐  ┌──────▼───────┐
         │ OpenTelemetry │  │ Redis  │  │  Prometheus  │
         │ Collectors    │  │Streams │  │   /metrics   │
         │ (Jaeger/Tempo)│  └────┬───┘  └──────┬───────┘
         └───────────────┘       │             │
                                 │             │
┌────────────────────────────────▼─────────────▼──────────────────────┐
│                         PROCESSING LAYER                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │             Metrics Processor (Worker Service)               │   │
│  │  • Redis Consumer Group (Horizontal Scaling)                 │   │
│  │  • Rolling Window Aggregation                                │   │
│  │  • Z-Score Anomaly Detection                                 │   │
│  │  • Alert Rule Evaluation                                     │   │
│  │  • TimescaleDB Persistence                                   │   │
│  └───────────┬──────────────────────────────┬───────────────────┘   │
│              │                              │                       │
│  ┌───────────▼────────────┐    ┌───────────▼──────────────┐        │
│  │   Alert Engine         │    │   Storage Layer          │        │
│  │  • YAML Rule Engine    │    │   TimescaleDB           │        │
│  │  • Cooldown Management │    │  • Hypertables          │        │
│  │  • Slack Integration   │    │  • Sessions/Turns       │        │
│  │  • PagerDuty Events    │    │  • Event Tables         │        │
│  └────────────────────────┘    └──────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                  ┌──────────────┼──────────────┐
                  │              │              │
         ┌────────▼──────┐  ┌───▼────┐  ┌──────▼───────┐
         │   Grafana     │  │Streamlit│ │Slack/PagerDuty│
         │(Ops Dashboard)│  │Analytics│ │   (Alerts)    │
         └───────────────┘  └────────┘  └───────────────┘
```

### Component Responsibilities

#### 1. Collection Layer (SDK)
- **VoiceMonCollector**: Central telemetry hub, manages sessions/turns, coordinates exporters
- **Integrations**: Framework-specific adapters (LiveKit, Pipecat, Vapi)
- **Exporters**: Pluggable output destinations (Console, Redis, Prometheus, OTel)
- **Data Models**: Pydantic v2 schemas enforcing 4-layer framework

#### 2. Processing Layer (Worker)
- **Metrics Processor**: Async Redis consumer with consumer groups for horizontal scaling
- **Alert Engine**: YAML-driven rule evaluation with cooldown management
- **Anomaly Detection**: Z-score based statistical anomaly detection
- **Storage**: TimescaleDB async client for time-series + relational data

#### 3. Visualization Layer
- **Grafana**: Real-time operational dashboards with Prometheus + TimescaleDB datasources
- **Streamlit**: Analytical dashboard with call replay, drift detection, custom queries

#### 4. Alerting Layer
- **Slack Notifier**: Block Kit formatted alerts with severity-based channel routing
- **PagerDuty Integration**: Events API v2 for incident management (P0/P1 only)

---

## Low-Level Design (LLD)

### 1. Core Data Models (`voicemon/core/models.py`)

#### Entity Hierarchy
```
VoiceSession (1)
    ├── Turn (N)
    │   ├── STTEvent (0..N)
    │   ├── LLMEvent (0..N)
    │   │   └── ToolCallEvent (0..N)
    │   ├── TTSEvent (0..N)
    │   ├── InfraEvent (0..N)
    │   └── UXEvent (0..N)
    └── OutcomeEvent (0..1)
```

#### Key Models

**VoiceSession**
```python
class VoiceSession(BaseModel):
    session_id: str           # UUID hex
    agent_id: str             # Agent identifier
    agent_version: str        # Version/build number
    framework: str            # livekit|pipecat|vapi
    started_at: datetime      # UTC timestamp
    ended_at: datetime | None
    status: SessionStatus     # active|completed|failed|timeout
    duration_ms: float | None # Computed on end
    total_turns: int          # Aggregated count
    total_cost_usd: float | None
    metadata: dict[str, Any]
    tags: list[str]
```

**Turn**
```python
class Turn(BaseModel):
    turn_id: str              # UUID hex
    session_id: str           # Parent session
    turn_number: int          # Sequential within session
    speaker: Speaker          # user|agent|system
    started_at: datetime
    ended_at: datetime | None
    duration_ms: float | None
    was_interrupted: bool
    transcript: str
    metadata: dict[str, Any]
```

**Layer 2 Events (Execution)**
- `STTEvent`: Speech-to-Text metrics (latency, confidence, transcript, WER)
- `LLMEvent`: Language Model metrics (TTFT, tokens, cost, tool calls)
- `TTSEvent`: Text-to-Speech metrics (TTFB, audio duration, cost)

**Layer 3 Event (UX)**
- `UXEvent`: User experience metrics (E2E latency, interruptions, silence, sentiment)

**Layer 4 Event (Outcome)**
- `OutcomeEvent`: Business outcome metrics (task completion, escalation, compliance)

### 2. Collector Architecture (`voicemon/core/collector.py`)

#### Design Pattern: Hub-and-Spoke with Multi-Exporter

```python
class VoiceMonCollector:
    _exporters: list[BaseExporter]  # Plugin pattern
    _sessions: dict[str, VoiceSession]  # In-memory session cache
    _turns: dict[str, Turn]  # In-memory turn cache
    _turn_reports: dict[str, TurnReport]  # Aggregated turn data
    _lock: asyncio.Lock  # Thread-safety for async
```

#### Key Methods

**Session Lifecycle**
```python
def start_session(...) -> VoiceSession
    # Creates new session, initializes cache
    
async def end_session(...) -> VoiceSession
    # Computes duration, aggregates metrics
    # Emits OutcomeEvent + SessionReport to all exporters
```

**Turn Lifecycle**
```python
def start_turn(...) -> Turn
    # Creates turn, links to session
    # Initializes TurnReport aggregate
    
async def end_turn(...) -> Turn
    # Computes duration, finalizes transcript
    # Emits complete TurnReport to all exporters
```

**Event Recording (4-Layer)**
```python
async def record_stt(...) -> STTEvent
async def record_llm(...) -> LLMEvent
async def record_tts(...) -> TTSEvent
async def record_infra(...) -> InfraEvent
async def record_ux(...) -> UXEvent
async def record_outcome(...) -> OutcomeEvent
```

Each method:
1. Constructs event model with Pydantic validation
2. Updates internal TurnReport aggregate if turn_id provided
3. Emits to all registered exporters via `_emit()`
4. Handles exporter failures gracefully (logged, non-blocking)

### 3. Integration Layer (`voicemon/integrations/`)

#### LiveKit Integration (`livekit.py`)

**Hook Pattern**: Decorator-based event subscription
```python
def instrument_livekit(agent_session, collector, agent_id):
    session_id = collector.start_session(...)
    
    @agent_session.on("metrics_collected")
    async def _on_metrics(metrics):
        # Extract STT/LLM/TTS metrics from LiveKit
        collector.record_stt(...)
        collector.record_llm(...)
        collector.record_tts(...)
    
    @agent_session.on("user_state_changed")
    async def _on_user_state(state):
        # Track turn boundaries
        
    @agent_session.on("agent_speech_interrupted")
    async def _on_interruption():
        collector.record_ux(event_type="interruption")
```

**Design**: Non-invasive instrumentation via event hooks, zero agent code modification.

#### Pipecat Integration (`pipecat.py`)

**Observer Pattern**: Implements `BaseObserver` interface
```python
class VoiceMonPipecatObserver(BaseObserver):
    async def on_frame(self, frame: Frame):
        # Intercept pipeline frames
        if isinstance(frame, TranscriptionFrame):
            collector.record_stt(...)
        elif isinstance(frame, LLMResponseFrame):
            collector.record_llm(...)
        elif isinstance(frame, TTSAudioFrame):
            collector.record_tts(...)
```

**Design**: Frame-level interception for fine-grained visibility.

#### Vapi Integration (`vapi.py`)

**Two Modes**:

1. **Webhook Mode**: FastAPI router for real-time events
```python
def create_vapi_router(collector, webhook_secret):
    router = APIRouter()
    
    @router.post("/vapi/webhook")
    async def handle_vapi_webhook(event: VapiWebhookEvent):
        # Validate signature
        # Map Vapi event → VoiceMon event
        if event.type == "transcript":
            collector.record_stt(...)
```

2. **Polling Mode**: REST client for historical data
```python
class VapiClient:
    async def poll_recent_calls(minutes: int):
        # Query Vapi API for recent call IDs
        # Fetch call details
        # Backfill VoiceMon events
```

### 4. Exporter Architecture (`voicemon/exporters/`)

#### Base Exporter Pattern
```python
class BaseExporter(ABC):
    @abstractmethod
    async def export(self, stream: str, event: dict[str, Any]) -> None:
        pass
    
    async def flush(self) -> None:
        pass  # Optional batch flush
    
    async def close(self) -> None:
        pass  # Cleanup resources
```

#### Redis Streams Exporter (`redis.py`)

**Design**: Lightweight, at-least-once delivery, consumer groups

```python
class RedisStreamsExporter(BaseExporter):
    async def export(self, event_type: str, data: dict):
        stream = f"{prefix}:{event_type}"
        await redis.xadd(stream, {
            "event_type": event_type,
            "timestamp": utcnow().isoformat(),
            "data": json.dumps(data)
        }, maxlen=100_000, approximate=True)
```

**Key Features**:
- Consumer groups for horizontal worker scaling
- Bounded stream size (100k messages, LRU eviction)
- Pipeline support for batch exports
- ACK mechanism for reliable processing

#### Prometheus Exporter (`prometheus.py`)

**Design**: Histogram-based latency tracking, counter-based event tracking

```python
class PrometheusExporter(BaseExporter):
    _histograms: dict[str, Histogram]  # Latency metrics
    _counters: dict[str, Counter]      # Event counts
    _gauges: dict[str, Gauge]          # Current values
    
    async def export(self, event_type: str, data: dict):
        if event_type == "stt":
            self._histograms["stt_latency"].observe(data["latency_ms"])
            self._counters["stt_total"].inc()
        # ... similar for llm, tts, ux
```

**Exposed Metrics**:
- `voicemon_stt_latency_milliseconds` (histogram)
- `voicemon_llm_ttft_milliseconds` (histogram)
- `voicemon_tts_ttfb_milliseconds` (histogram)
- `voicemon_e2e_latency_milliseconds` (histogram)
- `voicemon_events_total{type="stt|llm|tts|ux"}` (counter)

### 5. Worker Processing Pipeline (`voicemon/workers/processor.py`)

#### Architecture: Multi-Stream Async Consumer

```python
class MetricsProcessor:
    EVENT_TYPES = ("session", "turn", "infra", "stt", "llm", "tts", "ux", "outcome")
    
    async def start(self):
        # Spawn consumer task per event type
        tasks = [
            asyncio.create_task(self._consume_loop(event_type))
            for event_type in self.EVENT_TYPES
        ]
        await asyncio.gather(*tasks)
```

#### Processing Pipeline (Per Event)

```
1. Read from Redis Stream (xreadgroup)
   ├─ Consumer group: voicemon-workers
   ├─ Consumer name: worker-1 (unique per pod/process)
   └─ Block: 2000ms, Batch: 100 messages

2. Process Each Message
   ├─ Persist to TimescaleDB
   ├─ Update Rolling Windows (WINDOW_SIZE=100)
   ├─ Check Thresholds
   │   ├─ Static thresholds (config.thresholds)
   │   └─ Anomaly detection (Z-score > 3.0)
   └─ Fire Alerts (with cooldown)

3. Acknowledge Messages (xack)
   └─ Only after successful processing
```

#### Rolling Window Aggregation

```python
_windows: dict[str, deque[float]] = defaultdict(lambda: deque(maxlen=100))

def _update_windows(event_type, data):
    metric_keys = {
        "stt": [("latency_ms", "stt_latency"), ("confidence", "stt_confidence")],
        "llm": [("ttft_ms", "llm_ttft")],
        "tts": [("ttfb_ms", "tts_ttfb")],
        "ux": [("e2e_latency_ms", "e2e_latency")],
    }
    for data_key, window_key in metric_keys.get(event_type, []):
        val = data.get(data_key)
        if val is not None and val > 0:
            self._windows[window_key].append(float(val))
```

**Real-time Statistics**:
- Mean, Median, P95, P99, Min, Max
- Used for dashboard queries and anomaly baseline

#### Anomaly Detection: Z-Score Method

```python
def _detect_anomaly(event_type, data):
    window = self._windows.get(window_key)
    if len(window) < ANOMALY_WINDOW:  # Need 50 samples
        return None
    
    values = list(window)
    current = values[-1]
    mean = statistics.mean(values[:-1])
    stdev = statistics.stdev(values[:-1])
    
    z_score = (current - mean) / stdev
    if abs(z_score) > Z_SCORE_THRESHOLD:  # 3.0 sigma
        return create_alert("anomaly", ...)
```

**Advantages**:
- No ML dependencies
- Real-time detection
- Adaptive to workload patterns
- Catches sudden spikes/drops

### 6. Alert Engine (`voicemon/alerts/engine.py`)

#### Rule Evaluation Pipeline

```
1. Load YAML Rules (alert_rules.yaml)
   ├─ Parse into AlertRule objects
   ├─ Validate metric names, operators, thresholds
   └─ Cache in memory

2. Evaluate Metric Against Rules
   ├─ Filter rules matching metric_name
   ├─ Apply operator (>, <, >=, <=, ==)
   └─ Check threshold breach

3. Cooldown Management
   ├─ Key: f"{rule_name}:{agent_id}"
   ├─ Track last fire time
   └─ Skip if within cooldown window

4. Notify
   ├─ Log to worker logs (always)
   ├─ Slack (all severities)
   ├─ PagerDuty (critical, p0 only)
   └─ Persist alert to TimescaleDB
```

#### Alert Rule Schema

```yaml
rules:
  - name: stt_latency_critical
    description: "STT latency critical — users hear silence"
    metric: stt_latency_ms
    operator: ">"
    threshold: 1000
    severity: critical
    cooldown_minutes: 3
    notify: [slack, pagerduty]
    tags: [stt, latency]
```

#### Notification Routing

| Severity | Slack | PagerDuty | Typical Use Case |
|----------|-------|-----------|------------------|
| info     | ✅    | ❌        | FYI events, low-priority trends |
| warning  | ✅    | ❌        | Approaching thresholds, investigate |
| critical | ✅    | ✅        | SLA breach imminent, action required |
| p0       | ✅    | ✅        | Production outage, page on-call |

### 7. Storage Layer (`voicemon/storage/`)

#### TimescaleDB Schema Design

**Hypertables (Time-Series Optimized)**:
- `infra_events` — Partitioned by `time`
- `stt_events` — Partitioned by `time`
- `llm_events` — Partitioned by `time`
- `tts_events` — Partitioned by `time`
- `ux_events` — Partitioned by `time`

**Regular Tables (Relational)**:
- `sessions` — Parent entity, indexed by `agent_id`, `started_at`, `status`
- `turns` — Child of sessions, indexed by `session_id`, `started_at`
- `alerts` — Alert history for audit trail

**Why TimescaleDB?**
1. **Relational + Time-Series**: Need JOINs (session→turns→events) + time-based queries
2. **Automatic Partitioning**: Hypertables auto-partition by time, optimized for recent data
3. **Continuous Aggregates**: Pre-computed rollups (hourly, daily) for fast dashboard queries
4. **Data Retention**: Automatic old data deletion via retention policies
5. **PostgreSQL Compatibility**: Standard SQL, rich ecosystem, familiar tooling

#### Retention Strategy

```sql
-- Raw events: 30 days
SELECT add_retention_policy('stt_events', INTERVAL '30 days');
SELECT add_retention_policy('llm_events', INTERVAL '30 days');

-- Aggregates: 365 days
CREATE MATERIALIZED VIEW stt_hourly_stats
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) AS bucket,
       agent_id,
       AVG(latency_ms) AS avg_latency,
       percentile_cont(0.95) WITHIN GROUP (ORDER BY latency_ms) AS p95_latency,
       COUNT(*) AS event_count
FROM stt_events
GROUP BY bucket, agent_id;

SELECT add_retention_policy('stt_hourly_stats', INTERVAL '365 days');
```

---

## System Design Patterns

### 1. **Multi-Exporter Pattern**
- **Problem**: Different consumers need telemetry in different formats
- **Solution**: Plugin-based exporter architecture
- **Benefits**: Add new destinations without modifying collector, parallel export

### 2. **Consumer Group Pattern (Redis Streams)**
- **Problem**: Single worker can't handle high event volume
- **Solution**: Redis consumer groups for horizontal scaling
- **Benefits**: Multiple workers share load, at-least-once delivery, failure recovery

### 3. **Rolling Window Aggregation**
- **Problem**: Need real-time statistics without full dataset scans
- **Solution**: Fixed-size deques (100 samples) with O(1) append/evict
- **Benefits**: Constant memory, fast percentile computation, no latency spikes

### 4. **Anomaly Detection: Z-Score**
- **Problem**: Static thresholds miss context-dependent issues
- **Solution**: Statistical outlier detection (z-score > 3σ)
- **Benefits**: Adaptive to workload, catches unusual patterns, no ML overhead

### 5. **Alert Cooldown**
- **Problem**: Alert storms from repeated threshold breaches
- **Solution**: Per-rule, per-agent cooldown tracking (default 5min)
- **Benefits**: Prevents spam, maintains signal-to-noise ratio

### 6. **Instrumentation Hooks (Observer Pattern)**
- **Problem**: Voice framework APIs differ (LiveKit vs Pipecat vs Vapi)
- **Solution**: Framework-specific adapters implementing common collector interface
- **Benefits**: Portable telemetry logic, minimal agent code changes

### 7. **Hypertable Partitioning**
- **Problem**: Time-series queries slow on large tables
- **Solution**: TimescaleDB automatic partitioning by time
- **Benefits**: Recent data fast, old data auto-evicted, write throughput scales

---

## Data Flow

### 1. Telemetry Collection Flow

```
Voice Agent (LiveKit/Pipecat/Vapi)
    │
    ├─ Event: User starts speaking
    │   └─→ collector.start_turn(speaker="user")
    │
    ├─ Event: STT transcript received
    │   └─→ collector.record_stt(transcript="hello", confidence=0.95, latency_ms=120)
    │       └─→ Exporters: [Console, Redis, Prometheus]
    │           ├─→ Console: Print to stdout
    │           ├─→ Redis: XADD voicemon:stt {...}
    │           └─→ Prometheus: stt_latency_histogram.observe(120)
    │
    ├─ Event: LLM response starts
    │   └─→ collector.record_llm(model="gpt-4", ttft_ms=450, tokens=50)
    │       └─→ Exporters emit to all destinations
    │
    ├─ Event: TTS audio starts
    │   └─→ collector.record_tts(provider="elevenlabs", ttfb_ms=180)
    │       └─→ Exporters emit to all destinations
    │
    ├─ Event: User hears first word
    │   └─→ collector.record_ux(e2e_latency_ms=750, ttfw_ms=650)
    │       └─→ Exporters emit to all destinations
    │
    └─ Event: Turn ends
        └─→ collector.end_turn(turn_id, transcript="hello world")
            └─→ Emit complete TurnReport with all aggregated events
```

### 2. Processing Flow (Worker)

```
Redis Streams (voicemon:stt)
    │
    ├─ XREADGROUP voicemon-workers worker-1
    │   └─→ [{id: "1234-0", data: {...}}]
    │
    └─→ MetricsProcessor._process_event()
        │
        ├─ 1. Persist to TimescaleDB
        │   └─→ INSERT INTO stt_events (session_id, latency_ms, ...) VALUES (...)
        │
        ├─ 2. Update Rolling Windows
        │   └─→ _windows["stt_latency"].append(120)  # deque
        │
        ├─ 3. Check Thresholds
        │   ├─→ Static: if latency > 1000ms → alert
        │   └─→ Anomaly: if z_score > 3.0 → alert
        │
        ├─ 4. Fire Alerts (if threshold breached)
        │   ├─→ Check cooldown (skip if recent)
        │   ├─→ AlertEngine.notify(alert)
        │   │   ├─→ Slack webhook (Block Kit message)
        │   │   └─→ PagerDuty (if severity=critical)
        │   └─→ INSERT INTO alerts (...) — audit trail
        │
        └─ 5. ACK Message
            └─→ XACK voicemon:stt voicemon-workers 1234-0
```

### 3. Query Flow (Dashboards)

**Grafana (Operational)**:
```
Grafana Dashboard
    │
    ├─ Prometheus Datasource
    │   └─→ Query: rate(voicemon_stt_latency_milliseconds_sum[5m]) / 
    │              rate(voicemon_stt_latency_milliseconds_count[5m])
    │       └─→ Result: Average STT latency over 5-minute windows
    │
    └─ TimescaleDB Datasource
        └─→ Query: SELECT time_bucket('1 minute', time) AS bucket,
                    percentile_cont(0.95) WITHIN GROUP (ORDER BY latency_ms) AS p95
                  FROM stt_events WHERE time > NOW() - INTERVAL '1 hour'
                  GROUP BY bucket ORDER BY bucket;
        └─→ Result: P95 latency per minute for last hour
```

**Streamlit (Analytical)**:
```
Streamlit App
    │
    └─→ TimescaleDB Query
        ├─→ Session List: SELECT * FROM sessions WHERE agent_id = ? ORDER BY started_at DESC LIMIT 100
        ├─→ Turn Drill-Down: SELECT * FROM turns WHERE session_id = ? ORDER BY turn_number
        ├─→ Event Timeline: 
        │    SELECT time, 'STT' AS type, latency_ms FROM stt_events WHERE session_id = ?
        │    UNION ALL
        │    SELECT time, 'LLM', ttft_ms FROM llm_events WHERE session_id = ?
        │    UNION ALL
        │    SELECT time, 'TTS', ttfb_ms FROM tts_events WHERE session_id = ?
        │    ORDER BY time;
        └─→ Aggregate Stats:
             SELECT agent_id,
                    AVG(duration_ms) AS avg_session_duration,
                    COUNT(CASE WHEN status = 'completed' THEN 1 END) / COUNT(*)::float AS success_rate
             FROM sessions WHERE started_at > NOW() - INTERVAL '7 days'
             GROUP BY agent_id;
```

---

## Deployment Architecture

### Docker Compose Stack (Development/Self-Hosted)

```yaml
services:
  timescale:     # PostgreSQL + TimescaleDB
  redis:         # Redis Streams
  prometheus:    # Metrics collection
  grafana:       # Ops dashboard
  worker:        # Metrics processor (Python)
  streamlit:     # Analytics dashboard (Python)
```

**Networking**:
- Timescale: 5432 (internal), 5432 (host) — for direct SQL access
- Redis: 6379 (internal), 6379 (host) — for SDK connections
- Prometheus: 9090 (internal), 9091 (host) — query interface
- Grafana: 3000 (both) — web UI
- Streamlit: 8501 (both) — web UI

**Data Volumes**:
- `timescale_data`: PostgreSQL data directory (persist sessions/events)
- `redis_data`: RDB/AOF snapshots (optional persistence)
- `prometheus_data`: TSDB blocks (30-day retention)
- `grafana_data`: Dashboard configs, user data

### Production Deployment (Kubernetes)

**Recommended Architecture**:

```
┌─────────────────────────────────────────────────────────────┐
│                         Kubernetes Cluster                  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  VoiceMon Worker Deployment                         │    │
│  │  • Replicas: 3-5 (horizontal scaling)               │    │
│  │  • Resource Requests: 500m CPU, 512Mi RAM           │    │
│  │  • Resource Limits: 2 CPU, 2Gi RAM                  │    │
│  │  • HPA: Scale on CPU > 70% or queue depth > 1000    │    │
│  │  • Consumer Name: Unique per pod (pod UID)          │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  TimescaleDB StatefulSet                            │    │
│  │  • Replicas: 1 primary + 2 read replicas           │    │
│  │  • Storage: 500Gi SSD (PVC with snapshot policy)   │    │
│  │  • Connection Pooling: PgBouncer sidecar           │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Redis StatefulSet                                  │    │
│  │  • Replicas: 1 primary + 2 replicas (Sentinel)     │    │
│  │  • Memory: 4Gi (maxmemory with LRU eviction)       │    │
│  │  • Persistence: AOF enabled                         │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Grafana Deployment                                 │    │
│  │  • Replicas: 2 (stateless, behind load balancer)   │    │
│  │  • External Access: Ingress with TLS                │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Streamlit Deployment                               │    │
│  │  • Replicas: 1 (stateful session, sticky routing)  │    │
│  │  • External Access: Ingress with TLS                │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

**Scaling Strategy**:

1. **Worker Scaling** (Horizontal):
   - HPA based on Redis stream depth: `kubectl get hpa voicemon-worker`
   - Target: Queue depth < 1000 messages per stream
   - Max replicas: 10 (ensure unique consumer names)

2. **Database Scaling** (Vertical + Read Replicas):
   - Primary: Handle writes (INSERTs)
   - Read replicas: Handle dashboard queries (SELECTs)
   - Connection pooling: PgBouncer (transaction mode)

3. **Redis Scaling** (Vertical):
   - Memory-bound (100k messages × 8 streams × 2KB avg = ~1.6GB)
   - Use Redis Cluster for >10k events/sec sustained

**High Availability**:
- **Worker**: Stateless, auto-restart on failure, consumer group ensures no message loss
- **TimescaleDB**: Primary + replicas, automatic failover (Patroni/Stolon)
- **Redis**: Sentinel for automatic primary election
- **Prometheus**: Remote write to long-term storage (Thanos/Cortex)

**Monitoring the Monitoring**:
- Worker health: `GET /health` endpoint, liveness probe
- Redis stream depth: `prometheus_query(redis_stream_length{stream="voicemon:stt"})`
- Database lag: `SELECT NOW() - MAX(time) FROM stt_events`
- Alert delivery: `SELECT COUNT(*) FROM alerts WHERE created_at > NOW() - INTERVAL '5 minutes'`

---

## Key Design Decisions

| Decision | Choice | Why | Trade-offs |
|----------|--------|-----|-----------|
| **Primary Database** | TimescaleDB | Voice data is relational (sessions→turns→events need JOINs) + time-series optimized | More complex than pure time-series DB (InfluxDB), but richer query capabilities |
| **Ingestion Pipeline** | Redis Streams | Lightweight, consumer groups for horizontal scaling, at-least-once delivery | Not as feature-rich as Kafka, but simpler operations |
| **Metrics Store** | Prometheus | Industry standard for SLOs, native histogram quantiles, Alertmanager integration | Pull-based model (requires /metrics endpoint), 30-day retention |
| **Ops Dashboard** | Grafana | Universal, supports both Prometheus + TimescaleDB datasources | Steeper learning curve than Metabase/Superset |
| **Analytics Dashboard** | Streamlit | Python-native, custom call replay + drift detection UX, fast iteration | Single-user sessions (not multi-tenant), requires sticky routing |
| **Anomaly Detection** | Z-score | Simple, no ML dependencies, catches latency spikes in real-time | Assumes normal distribution, doesn't handle seasonality |
| **Alert Routing** | Slack + PagerDuty | Slack for ChatOps, PagerDuty for incident management (P0/P1 escalation) | Requires separate integrations, no unified alerting UI |
| **SDK Language** | Python | Voice AI ecosystem (LiveKit, Pipecat, Vapi) is Python-first | Less performant than Go/Rust, but better ecosystem fit |
| **Data Models** | Pydantic v2 | Runtime validation, auto-serialization, IDE autocomplete | Slight overhead vs plain dataclasses, but catches bugs early |

---

## Performance Characteristics

### Throughput
- **Single Worker**: ~1,000 events/sec (mixed workload)
- **Worker Pod (3 replicas)**: ~3,000 events/sec
- **Redis Streams**: ~50,000 events/sec (write capacity)
- **TimescaleDB**: ~10,000 INSERTs/sec (hypertable, 16-core instance)

### Latency
- **SDK Overhead**: <1ms (in-memory, async export)
- **Redis Export**: <5ms P95 (local network)
- **End-to-End (Event → Alert)**: <2 seconds P95
- **Dashboard Query (hourly aggregate)**: <100ms P95

### Resource Usage (per Worker Pod)
- **CPU**: 200-500m (idle to moderate load)
- **Memory**: 256-512Mi (100k rolling window samples)
- **Network**: <10 Mbps (Redis read/write)

### Storage Growth
- **Raw Events**: ~2KB per event × 1M events/day = **2GB/day**
- **TimescaleDB Compression**: ~70% reduction after 1 day = **0.6GB/day**
- **30-Day Retention**: ~18GB uncompressed, ~5GB compressed
- **Continuous Aggregates**: Negligible (pre-computed rollups)

---

## Security Considerations

### 1. **Authentication & Authorization**
- Grafana: OAuth2/SAML integration recommended for production
- Streamlit: Basic auth or reverse proxy (nginx) with SSO
- TimescaleDB: TLS connections, role-based access control (RBAC)
- Redis: Password authentication, TLS in production

### 2. **Data Privacy**
- **PII Handling**: Transcripts may contain PII, use column-level encryption if needed
- **Retention**: Automatic deletion after 30 days (raw events)
- **Access Logs**: Audit trail for query access (PostgreSQL logging)

### 3. **Network Security**
- Internal services: Private network (Kubernetes ClusterIP, no external exposure)
- External dashboards: TLS termination at ingress, firewall rules
- Redis: Not exposed to internet, internal VPC only

### 4. **Secrets Management**
- Slack webhook URL: Kubernetes Secret, mounted as env var
- PagerDuty key: Kubernetes Secret, mounted as env var
- Database credentials: Kubernetes Secret, rotated quarterly

---

## Operational Runbook

### Deployment
```bash
# Development (Docker Compose)
docker compose up -d
docker compose exec timescale psql -U voicemon -d voicemon -f /schema/schema.sql

# Production (Kubernetes)
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secrets.yaml
kubectl apply -f k8s/timescale-statefulset.yaml
kubectl apply -f k8s/redis-statefulset.yaml
kubectl apply -f k8s/worker-deployment.yaml
kubectl apply -f k8s/grafana-deployment.yaml
```

### Monitoring
```bash
# Worker health
kubectl logs -f deployment/voicemon-worker

# Redis stream depth
kubectl exec -it redis-0 -- redis-cli XLEN voicemon:stt

# Database health
kubectl exec -it timescale-0 -- psql -U voicemon -c "SELECT COUNT(*) FROM stt_events WHERE time > NOW() - INTERVAL '5 minutes';"

# Alert history
kubectl exec -it timescale-0 -- psql -U voicemon -c "SELECT * FROM alerts ORDER BY created_at DESC LIMIT 10;"
```

### Troubleshooting

**Issue: Worker not processing events**
- Check Redis connection: `kubectl exec -it worker-0 -- redis-cli -u $REDIS_URL PING`
- Check consumer lag: `kubectl exec -it redis-0 -- redis-cli XPENDING voicemon:stt voicemon-workers`
- Check worker logs: `kubectl logs -f deployment/voicemon-worker --tail=100`

**Issue: Database query slow**
- Check active queries: `SELECT * FROM pg_stat_activity WHERE state = 'active' AND query_start < NOW() - INTERVAL '10 seconds';`
- Check missing indexes: `SELECT * FROM pg_stat_user_tables WHERE seq_scan > 1000;`
- Check hypertable health: `SELECT * FROM timescaledb_information.hypertables;`

**Issue: Alerts not firing**
- Check rule evaluation: `kubectl logs -f deployment/voicemon-worker | grep ALERT`
- Check Slack webhook: `curl -X POST $SLACK_WEBHOOK_URL -d '{"text":"test"}'`
- Check PagerDuty key: `curl -X POST https://events.pagerduty.com/v2/enqueue -H 'Content-Type: application/json' -d '{"routing_key":"'$PD_KEY'","event_action":"trigger","payload":{...}}'`

---

## Future Enhancements

1. **Machine Learning Integration**
   - LSTM-based anomaly detection for time-series forecasting
   - Drift detection for STT confidence degradation
   - Cost optimization recommendations (model switching)

2. **Advanced Tracing**
   - OpenTelemetry-native distributed tracing
   - Span correlation across STT→LLM→TTS pipeline
   - Flame graphs for latency waterfall

3. **Multi-Tenancy**
   - Tenant-level data isolation (PostgreSQL RLS)
   - Per-tenant dashboards and alert rules
   - Usage-based billing integration

4. **Real-Time Dashboards**
   - WebSocket-based live dashboards (replace Grafana auto-refresh)
   - Call-level replay with audio playback
   - Live session monitoring (in-progress calls)

5. **Cost Optimization**
   - Automatic model selection (GPT-4 → GPT-3.5 for simple queries)
   - TTS voice cloning cost analysis
   - Token usage trending and forecasting

---

## Conclusion

VoiceMon is designed as a **production-ready, horizontally-scalable observability platform** for voice AI pipelines. It combines:

- **Comprehensive Telemetry**: 4-layer framework captures infrastructure, execution, UX, and outcome metrics
- **Flexible Architecture**: Plugin-based exporters, multi-framework integrations, polyglot dashboards
- **Operational Excellence**: Real-time alerting, anomaly detection, runbook-driven troubleshooting
- **Cost-Effective**: Open-source components, self-hosted option, predictable resource usage

The architecture balances **simplicity** (Redis Streams, Z-score anomalies) with **power** (TimescaleDB, Grafana), making it accessible to small teams while scalable to enterprise deployments.

For questions or contributions, see [CONTRIBUTING.md](CONTRIBUTING.md).
