# VoiceMon Architecture Guide

> A complete walkthrough of VoiceMon's High-Level Design (HLD), Low-Level Design (LLD), every component, and how they all synchronize into a production-grade voice AI observability platform.

---

## Table of Contents

1. [What VoiceMon Does](#1-what-voicemon-does)
2. [High-Level Design (HLD)](#2-high-level-design-hld)
3. [Low-Level Design (LLD)](#3-low-level-design-lld)
4. [Component-by-Component Walkthrough](#4-component-by-component-walkthrough)
5. [How Everything Syncs Together](#5-how-everything-syncs-together)
6. [Infrastructure & Deployment](#6-infrastructure--deployment)
7. [Configuration System](#7-configuration-system)
8. [Testing Strategy](#8-testing-strategy)

---

## 1. What VoiceMon Does

VoiceMon is a **production observability platform for voice AI agents**. It monitors voice pipelines built on LiveKit, Pipecat, or Vapi by capturing telemetry across four layers:

| Layer | What It Captures | Example Metrics |
|-------|-----------------|-----------------|
| **L1: Infrastructure** | Network and transport quality | Jitter, packet loss, MOS score, bitrate |
| **L2: Execution** | The STT → LLM → TTS pipeline | Per-stage latency, confidence, token counts, TTFT/TTFB |
| **L3: User Experience** | Conversation quality as perceived by users | End-to-end latency, interruptions, silence ratio |
| **L4: Outcome** | Business results | Task success rate, escalations, cost, compliance |

This framework is based on the [Hamming AI 4-Layer Voice Observability Framework](https://www.hamming.ai/blog/voice-agent-observability).

---

## 2. High-Level Design (HLD)

### 2.1 System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        YOUR VOICE AGENT                                │
│            (LiveKit Agents / Pipecat / Vapi)                           │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │              voicemon SDK (1-line integration)                  │   │
│   │                                                                 │   │
│   │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐       │   │
│   │  │  LiveKit      │   │  Pipecat     │   │  Vapi        │       │   │
│   │  │  Integration  │   │  Observer    │   │  Webhooks +  │       │   │
│   │  │  (hooks)      │   │  (frames)    │   │  REST Client │       │   │
│   │  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘       │   │
│   │         │                  │                   │                │   │
│   │         └──────────────────┼───────────────────┘                │   │
│   │                            ▼                                    │   │
│   │                  ┌──────────────────┐                           │   │
│   │                  │ VoiceMonCollector│  ◄── Central telemetry hub│   │
│   │                  │  (core/collector)│                           │   │
│   │                  └────────┬─────────┘                           │   │
│   │                           │                                     │   │
│   │              ┌────────────┼────────────┐                        │   │
│   │              ▼            ▼            ▼                        │   │
│   │     ┌──────────────┐ ┌────────┐ ┌────────────┐                 │   │
│   │     │ OTel Bridge  │ │Exporters│ │  Console   │                │   │
│   │     │ (core/otel)  │ │(Redis/ │ │  Exporter  │                 │   │
│   │     │              │ │Prom)   │ │  (debug)   │                 │   │
│   │     └──────┬───────┘ └───┬────┘ └────────────┘                 │   │
│   └────────────┼─────────────┼──────────────────────────────────────┘   │
│                │             │                                          │
└────────────────┼─────────────┼──────────────────────────────────────────┘
                 │             │
        ┌────────┘      ┌──────┴──────────────┐
        ▼               ▼                     ▼
  ┌──────────┐   ┌─────────────┐     ┌──────────────┐
  │ Jaeger / │   │ Redis       │     │ Prometheus   │
  │ Tempo    │   │ Streams     │     │ /metrics     │
  │ (traces) │   │ (events)    │     │ (SLOs)       │
  └──────────┘   └──────┬──────┘     └──────┬───────┘
                        │                   │
                        ▼                   ▼
                 ┌─────────────┐     ┌──────────────┐
                 │ Worker      │     │ Grafana      │
                 │ Process     │     │ (Ops Dash)   │
                 │ (processor) │     └──────────────┘
                 └──────┬──────┘
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
       ┌────────────┐ ┌─────┐ ┌──────────┐
       │TimescaleDB │ │Alert│ │ Anomaly  │
       │(persistent │ │Engine│ │ Detection│
       │ storage)   │ │     │ │ (Z-score)│
       └──────┬─────┘ └──┬──┘ └──────────┘
              │          │
              ▼       ┌──┴──┐
       ┌──────────┐   ▼     ▼
       │Streamlit │ ┌─────┐ ┌──────────┐
       │Analytics │ │Slack│ │PagerDuty │
       │Dashboard │ │     │ │          │
       └──────────┘ └─────┘ └──────────┘
```

### 2.2 Data Flow Summary

The system has three distinct data paths:

1. **Tracing Path** (OpenTelemetry): `Voice Agent → OTel Spans → Jaeger/Tempo` — Distributed tracing with voice-specific span hierarchies.

2. **Metrics Path** (Prometheus): `Voice Agent → Prometheus Exporter → Prometheus → Grafana` — Real-time SLO monitoring and operational dashboards.

3. **Event Path** (Redis → TimescaleDB): `Voice Agent → Redis Streams → Worker → TimescaleDB → Streamlit` — Persistent event storage, aggregation, anomaly detection, and analytical dashboards.

### 2.3 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Primary DB | TimescaleDB | Voice data is relational (sessions→turns→events need JOINs) + native time-series hypertables |
| Ingestion buffer | Redis Streams | Lightweight, consumer groups for horizontal scaling, at-least-once delivery |
| Real-time metrics | Prometheus | Industry standard for SLOs, native histogram quantiles, Alertmanager compatibility |
| Ops Dashboard | Grafana | Universal, supports both Prometheus and TimescaleDB as data sources |
| Analytics Dashboard | Streamlit | Python-native, enables custom call replay and drift detection UX |
| Anomaly Detection | Z-score | Simple, no ML dependencies, catches latency spikes in real-time |
| Data Models | Pydantic v2 | Type-safe, fast validation, JSON serialization out of the box |
| Async Runtime | asyncio | Matches the async nature of voice agent frameworks (LiveKit, Pipecat) |

---

## 3. Low-Level Design (LLD)

### 3.1 Package Structure

```
voicemon/
├── __init__.py              # Public API surface — re-exports key classes
├── core/                    # Core SDK — the heart of VoiceMon
│   ├── __init__.py
│   ├── models.py            # Pydantic v2 data models (4-layer framework)
│   ├── config.py            # Configuration with industry-standard thresholds
│   ├── collector.py         # Central telemetry hub (VoiceMonCollector)
│   └── otel.py              # OpenTelemetry bridge with voice-aware spans
├── integrations/            # Framework-specific adapters
│   ├── __init__.py
│   ├── livekit.py           # LiveKit AgentSession event hooks
│   ├── pipecat.py           # Pipecat BaseObserver frame interception
│   └── vapi.py              # Vapi webhooks + REST client
├── exporters/               # Output adapters (where data goes)
│   ├── __init__.py
│   ├── base.py              # Exporter interface (ABC) + ConsoleExporter
│   ├── redis.py             # Redis Streams with consumer groups
│   └── prometheus.py        # Prometheus histograms/counters/gauges
├── storage/                 # Persistent storage layer
│   ├── __init__.py
│   ├── schema.sql           # TimescaleDB schema with hypertables
│   └── timescale.py         # Async TimescaleDB client (asyncpg)
├── workers/                 # Background processing
│   ├── __init__.py
│   └── processor.py         # Redis consumer → aggregation → anomaly detection
├── alerts/                  # Alerting subsystem
│   ├── __init__.py
│   ├── engine.py            # YAML rule engine with cooldowns
│   ├── slack.py             # Slack Block Kit notifications
│   └── pagerduty.py         # PagerDuty Events API v2
└── dashboards/              # Visualization
    ├── __init__.py
    ├── streamlit_app.py     # Streamlit analytics dashboard
    └── grafana/
        ├── provisioning.yaml
        └── voicemon_dashboard.json  # Pre-built Grafana dashboard
```

### 3.2 Data Model Hierarchy

The data models in `core/models.py` follow the 4-layer framework with a structural envelope:

```
Layer 0 — Structural Envelope
├── VoiceSession              # Top-level container: one voice call = one session
│   ├── session_id (UUID)
│   ├── agent_id, agent_version, framework
│   ├── started_at, ended_at, duration_ms
│   ├── status (active/completed/failed/timeout)
│   ├── metadata (dict), tags (list)
│   └── total_turns, total_cost_usd
│
└── Turn                      # One conversational turn (user speaks → agent responds)
    ├── turn_id (UUID)
    ├── session_id (FK → VoiceSession)
    ├── turn_number, speaker (user/agent/system)
    ├── started_at, ended_at, duration_ms
    ├── was_interrupted, transcript
    └── metadata

Layer 1 — Infrastructure
└── InfraEvent                # Network/transport quality snapshot
    ├── jitter_ms, packet_loss_pct, mos_score
    ├── rtt_ms, audio_codec, sample_rate_hz, bitrate_kbps
    └── session_id, timestamp

Layer 2 — Execution (the STT → LLM → TTS pipeline)
├── STTEvent                  # Speech-to-Text recognition
│   ├── transcript, is_final, confidence, language
│   ├── latency_ms, audio_duration_ms
│   ├── provider, model (e.g., "deepgram", "nova-2")
│   └── word_error_rate, word_confidences
│
├── LLMEvent                  # Large Language Model inference
│   ├── model, provider (e.g., "openai", "gpt-4o")
│   ├── ttft_ms (time-to-first-token), duration_ms
│   ├── prompt_tokens, completion_tokens, total_tokens
│   ├── tokens_per_second, cached_tokens
│   ├── tool_calls: list[ToolCallEvent]
│   └── cancelled, prompt_compliant, cost_usd
│
├── TTSEvent                  # Text-to-Speech synthesis
│   ├── text, provider, voice_id
│   ├── ttfb_ms (time-to-first-byte), duration_ms
│   ├── audio_duration_ms, characters_count
│   └── streamed, cancelled, cost_usd
│
└── ToolCallEvent             # Function/tool calls within LLM
    ├── function_name, arguments, result
    ├── success, latency_ms, error
    └── timestamp

Layer 3 — User Experience
└── UXEvent                   # Perceived conversation quality
    ├── e2e_latency_ms, ttfw_ms (time-to-first-word)
    ├── interruption_count, false_interruption_count
    ├── was_barge_in, silence_duration_ms
    ├── talk_ratio, frustration_score, sentiment
    └── is_dialog_loop, repeated_prompt_count, fallback_triggered

Layer 4 — Outcome
└── OutcomeEvent              # Business/compliance results
    ├── task_completed, task_name, turns_to_complete
    ├── escalated, transferred, transfer_target
    ├── compliance_pass, compliance_violations
    ├── cost_usd, revenue_impact_usd
    └── ended_reason, containment_success
```

**Aggregate types** compose these into higher-level reports:

- **`TurnReport`**: Wraps one `Turn` with its associated `STTEvent`, `LLMEvent`, `TTSEvent`, `UXEvent`, and `InfraEvent`.
- **`SessionReport`**: Wraps one `VoiceSession` with a list of `TurnReport`s and an optional `OutcomeEvent`. Provides computed properties like `e2e_latency_p50`, `total_interruptions`, and `avg_stt_confidence`.

### 3.3 Class Relationship Diagram

```
                    ┌──────────────────┐
                    │  VoiceMonConfig  │
                    │  (Pydantic)      │
                    └────────┬─────────┘
                             │ configures
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
┌──────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│VoiceMonCollector │ │MetricsProcessor │ │  AlertEngine    │
│(core/collector)  │ │(workers/process)│ │(alerts/engine)  │
└────────┬─────────┘ └────────┬────────┘ └────────┬────────┘
         │                    │                    │
    uses │ exports to    reads│from            notifies via
         │                    │                    │
         ▼                    ▼               ┌────┴────┐
┌──────────────────┐  ┌─────────────┐         ▼         ▼
│  BaseExporter    │  │  Redis      │  ┌──────────┐ ┌────────┐
│  (abstract)      │  │  Streams    │  │  Slack   │ │Pager-  │
├──────────────────┤  └─────────────┘  │ Notifier │ │Duty    │
│ ConsoleExporter  │                   └──────────┘ └────────┘
│ RedisStreams     │  ┌─────────────┐
│  Exporter       │  │TimescaleStore│
│ Prometheus      │  │(storage/    │
│  Exporter       │  │ timescale)  │
└──────────────────┘  └─────────────┘
```

### 3.4 OpenTelemetry Span Hierarchy

The `VoiceTracer` in `core/otel.py` creates nested spans that model the voice pipeline:

```
voice_session (root span)
├── voice.session.id, voice.agent.id, voice.framework
│
├── voice_turn (child span — one per turn)
│   ├── voice.turn.number, voice.turn.speaker
│   │
│   ├── stt_recognition (child span)
│   │   ├── voice.stt.provider, voice.stt.model
│   │   ├── voice.stt.confidence, voice.stt.latency_ms
│   │   └── voice.stt.language
│   │
│   ├── llm_inference (child span)
│   │   ├── gen_ai.provider.name, gen_ai.request.model
│   │   ├── voice.llm.ttft_ms
│   │   ├── gen_ai.usage.input_tokens, gen_ai.usage.output_tokens
│   │   │
│   │   └── tool_call (child span — zero or more)
│   │       └── voice.tool.function_name
│   │
│   └── tts_synthesis (child span)
│       ├── voice.tts.provider, voice.tts.voice_id
│       └── voice.tts.ttfb_ms
│
└── (next voice_turn...)
```

Each span type uses context managers so errors are automatically recorded as span exceptions with `StatusCode.ERROR`.

---

## 4. Component-by-Component Walkthrough

### 4.1 `core/models.py` — Data Models

**Purpose**: Defines the canonical Pydantic v2 data models for all telemetry events.

**Key details**:
- Every model uses `Field(default_factory=...)` for mutable defaults (dicts, lists).
- UUIDs are generated via `uuid.uuid4().hex` (32-char hex strings, no dashes).
- Timestamps default to `datetime.now(timezone.utc)` — always UTC.
- The `Speaker` enum constrains turn speakers to `user`, `agent`, or `system`.
- The `SessionStatus` enum tracks lifecycle: `active` → `completed`/`failed`/`timeout`.
- `TurnReport` and `SessionReport` are aggregate types that compose events. `SessionReport` provides computed properties (`e2e_latency_p50`, `total_interruptions`, `avg_stt_confidence`) for quick analytics.

### 4.2 `core/config.py` — Configuration

**Purpose**: Centralized, type-safe configuration using Pydantic models.

**Key details**:
- `VoiceMonConfig` is the root config object, composing sub-configs for each subsystem:
  - `RedisConfig` — connection URL, stream prefix, consumer group, max stream length
  - `TimescaleConfig` — DSN, connection pool sizes, retention days
  - `PrometheusConfig` — enabled flag, port
  - `AlertConfig` — rules file path, Slack settings, PagerDuty settings
  - `OTelConfig` — service name, exporter type, endpoint
- Industry-standard latency thresholds are embedded as defaults in the `thresholds` dict.
- Supports loading from YAML via `VoiceMonConfig.from_yaml(path)`.

### 4.3 `core/collector.py` — VoiceMonCollector

**Purpose**: The **central hub** that all integrations report to. It manages session/turn lifecycle and dispatches events to exporters.

**How it works**:

```
Integration (LiveKit/Pipecat/Vapi)
        │
        │ calls collector.start_session()
        │ calls collector.record_stt()
        │ calls collector.record_llm()
        │ etc.
        ▼
┌──────────────────────────────────────┐
│        VoiceMonCollector             │
│                                      │
│  _sessions: dict[str, VoiceSession] │  ◄── In-memory session store
│  _turns: dict[str, Turn]            │  ◄── In-memory turn store
│  _turn_reports: dict[str, TurnReport]│ ◄── Assembles events per-turn
│  _session_turns: dict[str, list]    │  ◄── Tracks turn order per session
│  _exporters: list[BaseExporter]     │  ◄── Output destinations
│                                      │
│  async _emit(stream, event)         │  ◄── Fans out to all exporters
└──────────────────────────────────────┘
```

**Lifecycle methods**:
- `start_session()` → creates `VoiceSession`, returns session_id
- `start_turn()` → creates `Turn` + `TurnReport`, links to session
- `record_stt/llm/tts/ux/infra()` → creates event, attaches to `TurnReport`, emits to exporters
- `end_turn()` → finalizes turn timing, emits full `TurnReport`
- `end_session()` → finalizes session, creates `OutcomeEvent`, emits both

**Important**: The collector must be started with `await collector.start()` before events are emitted. Without starting, `_emit()` silently drops events. If no exporters are added, it defaults to `ConsoleExporter`.

### 4.4 `core/otel.py` — OpenTelemetry Bridge

**Purpose**: Creates voice-specific distributed traces using OpenTelemetry SDK.

**Key details**:
- `VoiceTracer` wraps the OTel `TracerProvider` and provides context managers for each span type.
- `VoiceAttributes` defines semantic attribute names following OTel conventions (e.g., `gen_ai.provider.name`, `gen_ai.request.model` for LLM spans).
- Supports both `BatchSpanProcessor` (production) and `SimpleSpanProcessor` (debugging).
- Each context manager catches exceptions and records them as span errors before re-raising.

### 4.5 `exporters/base.py` — Exporter Interface

**Purpose**: Defines the abstract `BaseExporter` protocol that all exporters implement.

**Interface**:
```python
class BaseExporter(ABC):
    async def export(self, stream: str, event: dict) -> None: ...
    async def flush(self) -> None: ...
    async def close(self) -> None: ...
```

**ConsoleExporter**: A debug exporter that prints events to stdout. In verbose mode, prints full JSON; in compact mode, prints just the event ID prefix.

### 4.6 `exporters/redis.py` — Redis Streams Exporter

**Purpose**: Publishes telemetry events to Redis Streams for asynchronous downstream processing.

**How it works**:
1. Creates 8 Redis Streams (one per event type: session, turn, infra, stt, llm, tts, ux, outcome).
2. Each stream has a consumer group (`voicemon-workers`) for horizontal scaling.
3. Events are serialized to JSON and published with `XADD`.
4. Streams are capped at `max_stream_len` (default 100K) using approximate trimming.

**Key methods**:
- `export()` / `export_async()` — Publish single event
- `export_batch()` — Pipeline multiple events for efficiency
- `read_stream()` — Consumer group read with blocking
- `ack()` — Acknowledge processed messages
- `get_pending_count()` / `get_stream_length()` — Monitoring helpers

### 4.7 `exporters/prometheus.py` — Prometheus Exporter

**Purpose**: Exposes voice pipeline metrics as Prometheus instruments for real-time SLO monitoring.

**Metric instruments created**:

| Type | Name | Labels | Purpose |
|------|------|--------|---------|
| Histogram | `voicemon_stt_latency_ms` | provider, model | STT recognition latency |
| Histogram | `voicemon_llm_ttft_ms` | provider, model | LLM time-to-first-token |
| Histogram | `voicemon_llm_total_ms` | provider, model | LLM total inference time |
| Histogram | `voicemon_tts_ttfb_ms` | provider | TTS time-to-first-byte |
| Histogram | `voicemon_tts_total_ms` | provider | TTS total synthesis time |
| Histogram | `voicemon_e2e_latency_ms` | agent_id | End-to-end latency |
| Histogram | `voicemon_stt_confidence` | provider | STT confidence distribution |
| Counter | `voicemon_sessions_total` | agent_id, framework | Total sessions |
| Counter | `voicemon_turns_total` | agent_id, speaker | Total turns |
| Counter | `voicemon_interruptions_total` | agent_id | Interruption events |
| Counter | `voicemon_errors_total` | agent_id, component | Pipeline errors |
| Counter | `voicemon_tool_calls_total` | agent_id, function_name | Tool/function calls |
| Counter | `voicemon_llm_tokens_total` | provider, direction | LLM token consumption |
| Counter | `voicemon_tasks_resolved_total` | agent_id, resolution_type | Task resolution |
| Gauge | `voicemon_active_sessions` | agent_id | Currently active sessions |
| Gauge | `voicemon_stt_confidence_current` | agent_id | Most recent STT confidence |
| Gauge | `voicemon_mos_score` | agent_id | Current Mean Opinion Score |

**Bucket boundaries**: Latency histograms use buckets tuned for voice: `(50, 100, 200, 300, 500, 800, 1000, 1500, 2000, 3000, 5000)` ms. Confidence histograms use: `(0.1, 0.2, ..., 0.95, 0.99)`.

**Event routing**: The `_event_handlers` dict maps event type strings to handler methods. Each handler extracts the relevant fields and updates the corresponding Prometheus instruments.

### 4.8 `integrations/livekit.py` — LiveKit Integration

**Purpose**: Hooks into LiveKit's `AgentSession` event system to automatically capture telemetry.

**How it works**:
```python
session_id = instrument_livekit(agent_session, collector, agent_id="my-agent")
```

This single function call:
1. Creates a VoiceMon session via `collector.start_session(framework="livekit")`.
2. Registers event handlers on the `AgentSession`:
   - `metrics_collected` → Records STT, LLM, TTS latencies from LiveKit's built-in metrics.
   - `user_state_changed` → Detects when user starts/stops speaking to manage turn boundaries.
   - `agent_state_changed` → Detects when agent starts/stops speaking to complete turns.
   - `user_input_transcribed` → Logs user transcripts.
   - `agent_speech_interrupted` → Records interruption UX events.
   - `function_tools_executed` → Records tool/function calls.
   - `close` → Ends the VoiceMon session.

**State tracking**: Uses a `_state` dict to track whether a turn is active and when speaking started. The `_ms()` helper converts seconds to milliseconds (LiveKit reports in seconds).

### 4.9 `integrations/pipecat.py` — Pipecat Integration

**Purpose**: Implements a Pipecat `BaseObserver` that intercepts frames flowing through the pipeline.

**How it works**:
```python
observer = VoiceMonPipecatObserver(collector, agent_id="my-agent")
pipeline = Pipeline([...], observers=[observer])
```

The observer receives callbacks:
- `on_pipeline_started()` → Creates VoiceMon session.
- `on_pipeline_stopped()` → Ends session.
- `on_push_frame()` / `on_pull_frame()` → Called on every frame. Dispatches to `_handle_frame()`.

**Frame dispatch**: The `_handle_frame()` method examines the frame class name and routes:

| Frame Name | Action |
|------------|--------|
| `UserStartedSpeakingFrame` | Start turn, begin STT timer |
| `TranscriptionFrame` / `InterimTranscriptionFrame` | Record STT event |
| `LLMFullResponseStartFrame` | Begin LLM timer |
| `LLMFullResponseEndFrame` / `TextFrame` | Record LLM event |
| `TTSStartedFrame` | Begin TTS timer |
| `TTSStoppedFrame` / `AudioRawFrame` | Record TTS event |
| `BotInterruptionFrame` etc. | Record interruption UX event |
| `BotStoppedSpeakingFrame` | End current turn |
| `MetricsFrame` | Ingest Pipecat's built-in metrics |

**Latency measurement**: Uses `time.monotonic()` to measure elapsed time between start/end frames for each pipeline stage.

### 4.10 `integrations/vapi.py` — Vapi Integration

**Purpose**: Supports two integration modes for Vapi voice agents.

**Mode 1 — Webhooks** (real-time):
```python
router = create_vapi_router(collector, webhook_secret="secret")
app.include_router(router)  # Adds POST /vapi/webhook endpoint
```

The `VapiWebhookHandler` processes Vapi's server-message webhook payloads:

| Webhook Type | Action |
|-------------|--------|
| `status-update` (in-progress) | Start VoiceMon session |
| `status-update` (ended) | End session |
| `transcript` | Record STT event |
| `speech-update` (started/stopped) | Start/end turns |
| `end-of-call-report` | Record outcome with cost, analysis |
| `tool-calls` | Record tool call events |
| `hang` | Record UX hang notification |

The handler maintains `_active_calls` mapping Vapi call IDs to VoiceMon session IDs. The `_ensure_session()` method handles out-of-order webhook delivery by creating sessions on-demand.

**Mode 2 — Polling** (batch):
```python
client = VapiClient(api_key="key", collector=collector)
session_ids = await client.poll_recent_calls(minutes=5)
```

Uses `httpx` to query Vapi's REST API (`GET /call`), then ingests each call's messages as turns and records outcomes with cost data.

### 4.11 `storage/schema.sql` — TimescaleDB Schema

**Purpose**: Defines the persistent storage schema optimized for time-series voice telemetry.

**Table structure**:
- **`sessions`** — Regular PostgreSQL table (UUID PK) with indexes on `agent_id`, `started_at`, `status`.
- **`turns`** — Regular table with FK to sessions, indexed on `session_id`.
- **`infra_events`**, **`stt_events`**, **`llm_events`**, **`tts_events`**, **`ux_events`**, **`outcome_events`** — TimescaleDB hypertables partitioned by `time` column. Each has indexes on `session_id` and relevant dimension columns.
- **`alerts`** — Regular table for persisting fired alerts.

**Continuous aggregates** (materialized views refreshed hourly):
- `hourly_latency_stats` — STT latency percentiles by provider
- `hourly_stt_stats` — STT confidence statistics by provider/model
- `hourly_llm_stats` — LLM TTFT percentiles and token counts by model

**Retention policies**: Raw events are automatically dropped after 30 days. Continuous aggregates are retained independently (typically 1 year).

**`session_summary` view**: Joins sessions with their aggregated STT/LLM/TTS latencies, turn counts, interruption counts, and task success — used by the Streamlit dashboard.

### 4.12 `storage/timescale.py` — TimescaleDB Client

**Purpose**: Async client for reading/writing to TimescaleDB using `asyncpg`.

**Key details**:
- Uses a connection pool (`asyncpg.Pool`) with configurable min/max sizes.
- `init_schema()` can run the bundled `schema.sql` or a custom schema file.
- Write methods (`insert_session`, `insert_turn`, `insert_stt_event`, etc.) use parameterized queries.
- Sessions use `ON CONFLICT ... DO UPDATE` for upsert semantics.
- Query methods provide `get_session()`, `get_recent_sessions()`, `get_session_turns()`, `get_latency_percentiles()`, and `get_stt_drift()`.
- The `_parse_ts()` helper handles timestamp parsing from strings or datetime objects.

### 4.13 `workers/processor.py` — Metrics Processor Worker

**Purpose**: The background worker that connects Redis Streams to TimescaleDB, performs aggregation, and fires alerts.

**Architecture**:
```
Redis Streams (8 event types)
        │
        │  xreadgroup (consumer group)
        ▼
┌──────────────────────────────────┐
│     MetricsProcessor             │
│                                  │
│  1. _persist() → TimescaleDB     │  Write to permanent storage
│  2. _update_windows() → rolling  │  Track rolling statistics
│  3. _check_thresholds() → alerts │  Evaluate alert rules
│     └─ _detect_anomaly()         │  Z-score anomaly detection
│                                  │
│  _stats_reporter() → logs        │  Log stats every 60s
└──────────────────────────────────┘
```

**Concurrency model**: Runs 8 concurrent `_consume_loop` tasks (one per event type) plus a stats reporter task, all managed by `asyncio.gather()`. Signal handlers (`SIGINT`, `SIGTERM`) trigger graceful shutdown.

**Rolling windows**: Uses `collections.deque(maxlen=100)` per metric to maintain a sliding window. Tracked metrics: `stt_latency`, `stt_confidence`, `llm_ttft`, `llm_total`, `tts_ttfb`, `tts_total`, `infra_jitter`, `infra_mos`, `e2e_latency`.

**Anomaly detection**: Z-score algorithm on the rolling window:
```
z_score = (current_value - mean(window)) / stdev(window)
if |z_score| > 3.0:  → fire anomaly alert
```
Requires at least 50 data points in the window before checking.

**Alert cooldowns**: Prevents alert spam by tracking the last fire time per `rule_name:agent_id` key. Alerts are suppressed if within the cooldown period.

### 4.14 `alerts/engine.py` — Alert Engine

**Purpose**: YAML-driven rule engine that evaluates metrics against thresholds and routes notifications.

**AlertRule**: Parsed from YAML config. Each rule specifies:
- `name`, `description` — Identity
- `metric`, `operator` (`>`, `<`, `>=`, `<=`, `==`), `threshold` — Condition
- `severity` (`info`, `warning`, `critical`, `p0`) — Priority
- `cooldown_minutes` — Suppress duplicate alerts
- `notify` (`slack`, `pagerduty`) — Routing targets
- `enabled`, `tags` — Control and categorization

**AlertEngine**:
- Loads rules from `alert_rules.yaml` (or falls back to 4 hardcoded defaults).
- `evaluate_metric(metric_name, value, context)` — Tests value against all matching rules, applies cooldowns, fires notifications.
- `notify(alert)` — Routes to Slack (all severities) and PagerDuty (critical/p0 only).

### 4.15 `alerts/slack.py` — Slack Notifier

**Purpose**: Posts rich Block Kit messages to Slack via incoming webhooks.

**Message format**:
- Header with severity emoji (ℹ️ / ⚠️ / 🚨 / 🔥)
- Fields section: severity, agent ID, value, threshold
- Message body with optional @mentions for critical/p0
- Context with session ID

**Severity-based routing**: Different Slack channels per severity (configurable via `SlackConfig.channels`).

### 4.16 `alerts/pagerduty.py` — PagerDuty Notifier

**Purpose**: Triggers, acknowledges, and resolves PagerDuty incidents via Events API v2.

**Key details**:
- `trigger_incident()` — Creates incident with dedup key `voicemon-{rule_name}-{agent_id}`.
- `acknowledge_incident()` / `resolve_incident()` — Lifecycle management.
- Maps VoiceMon severities to PagerDuty severities: `p0`/`critical` → `critical`, `warning` → `warning`, `info` → `info`.

### 4.17 `dashboards/streamlit_app.py` — Analytics Dashboard

**Purpose**: Rich analytical dashboard for deep-diving into voice agent performance.

**Tabs**:
1. **Overview** — KPI row (sessions, duration, turns, interruptions, confidence) + sessions over time chart + outcome distribution.
2. **Latency** — Pipeline latency budget breakdown (STT + LLM + TTS = Total) with P50/P95 and budget usage percentages. STT latency time series.
3. **Quality** — STT confidence trend with automatic drift detection (compares recent 4 buckets to baseline). Low-confidence transcript list.
4. **Sessions** — Session explorer with drill-down to turn-by-turn view, STT events, and LLM events for each session.
5. **Cost** — LLM token usage by model with estimated costs. Token usage trend over time.

**Data source**: Queries TimescaleDB directly via `psycopg2` with parameterized queries. Supports time range filtering (1h/6h/24h/7d/30d) and agent filtering.

### 4.18 `dashboards/grafana/` — Grafana Dashboard

**Purpose**: Pre-built operational dashboard for real-time monitoring.

**Panels** (all query Prometheus):
- **Row 1 — Health**: Active sessions, E2E latency P95, sessions/min, interruption rate, STT confidence, error rate
- **Row 2 — Latency**: STT latency distribution (P50/P95/P99), LLM TTFT distribution, TTS TTFB distribution, E2E latency over time with threshold lines, latency budget burn rate
- **Row 3 — Throughput**: Turns/min by speaker, LLM token consumption, STT confidence over time
- **Row 4 — Infrastructure**: MOS score gauge, task resolution pie chart, tool calls by function

**Provisioning**: Auto-loaded by Grafana via `provisioning.yaml` and datasource config in `grafana-datasources.yml`.

---

## 5. How Everything Syncs Together

### 5.1 End-to-End Data Flow (A Single Voice Turn)

Here's what happens when a user speaks one sentence to a LiveKit-based voice agent:

```
1. User speaks "What's my balance?"
   │
   ▼
2. LiveKit fires "user_state_changed: speaking"
   → instrument_livekit handler calls collector.start_turn(speaker="user")
   → Turn object created in collector._turns, TurnReport created
   │
   ▼
3. LiveKit STT processes audio, fires "metrics_collected" with stt_duration=0.15s
   → handler calls collector.record_stt(latency_ms=150, confidence=0.95, ...)
   → STTEvent created, attached to TurnReport
   → collector._emit("stt", event_data) fans out to all exporters:
     │
     ├── ConsoleExporter: prints "[voicemon:stt] abc12345"
     ├── RedisStreamsExporter: XADD voicemon:stt {data: "{...}"}
     └── PrometheusExporter: voicemon_stt_latency_ms.observe(150)
   │
   ▼
4. LLM generates response, LiveKit fires "metrics_collected" with llm_ttft=0.3s
   → handler calls collector.record_llm(ttft_ms=300, ...)
   → LLMEvent created, attached to TurnReport
   → emitted to all exporters (same pattern as step 3)
   │
   ▼
5. TTS synthesizes speech, LiveKit fires "metrics_collected" with tts_ttfb=0.12s
   → handler calls collector.record_tts(ttfb_ms=120, ...)
   → TTSEvent created, emitted to exporters
   │
   ▼
6. Agent speaks response, LiveKit fires "agent_state_changed: listening"
   → handler calls collector.end_turn()
   → Turn finalized (duration_ms calculated)
   → Full TurnReport emitted to exporters
   │
   ▼
7. Meanwhile, in the Worker process (separate container):
   │
   ├── Worker reads from Redis Stream "voicemon:stt" via consumer group
   │   → _process_event("stt", data):
   │     ├── _persist(): INSERT INTO stt_events VALUES(...)
   │     ├── _update_windows(): stt_latency deque.append(150)
   │     └── _check_thresholds(): 150ms < 500ms warning → no alert
   │
   ├── Worker reads from "voicemon:llm"
   │   → persists, updates window, checks thresholds
   │
   └── Worker reads from "voicemon:tts"
       → persists, updates window, checks thresholds
   │
   ▼
8. Prometheus scrapes /metrics from the worker every 10s
   → histogram_quantile(0.95, rate(voicemon_stt_latency_ms_bucket[5m]))
   → Grafana queries Prometheus, displays live latency charts
   │
   ▼
9. Streamlit dashboard queries TimescaleDB
   → SELECT percentile_cont(0.50) ... FROM stt_events WHERE time > NOW() - '1 hour'
   → Displays latency budget breakdown, confidence trends
```

### 5.2 Alert Flow (When Things Go Wrong)

```
1. STT latency spikes to 1200ms (e.g., Deepgram API degradation)
   │
   ▼
2. Worker processes the event:
   → _check_thresholds("stt", {latency_ms: 1200}):
     → 1200 > 1000 (stt_latency_critical_ms threshold) → build alert
   → _detect_anomaly("stt", data):
     → z_score = (1200 - 200) / 50 = 20.0 > 3.0 → build anomaly alert
   │
   ▼
3. _fire_alert(alert):
   → Check cooldown: first fire for this rule → proceed
   → Persist to TimescaleDB alerts table
   → AlertEngine.notify(alert):
     │
     ├── Slack: POST webhook with Block Kit message
     │   → 🚨 VoiceMon Alert: stt_latency_critical
     │   → Severity: CRITICAL | Agent: my-agent
     │   → Value: 1200.00 | Threshold: 1000.00
     │
     └── PagerDuty (severity=critical): POST Events API v2
         → Creates incident with dedup key
         → Pages on-call engineer
   │
   ▼
4. Subsequent 1200ms events within 3-minute cooldown → suppressed
```

### 5.3 Component Synchronization Points

| From | To | Mechanism | When |
|------|----|-----------|------|
| Integration → Collector | Method calls | `collector.record_stt()` etc. | Every pipeline event |
| Collector → Exporters | `_emit()` fan-out | `await exporter.export()` | After every record call |
| Redis Exporter → Worker | Redis Streams + Consumer Groups | `XREADGROUP` blocking read | Continuous polling |
| Worker → TimescaleDB | SQL `INSERT` via asyncpg | `_persist()` on each event | Every processed event |
| Worker → Alert Engine | Python method call | `_check_thresholds()` | Every processed event |
| Alert Engine → Slack/PD | HTTP POST (httpx) | `notify()` | On threshold breach |
| Prometheus Exporter → Prometheus | HTTP scrape (`/metrics`) | Pull-based, every 10s | Configured interval |
| Prometheus → Grafana | PromQL queries | Pull-based, dashboard refresh | Every 10s |
| TimescaleDB → Streamlit | SQL queries (psycopg2) | On page load/refresh | User interaction |

---

## 6. Infrastructure & Deployment

### 6.1 Docker Compose Stack

The `docker-compose.yml` defines 6 services:

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| `timescale` | `timescale/timescaledb:latest-pg16` | 5432 | Primary data store |
| `redis` | `redis:7-alpine` | 6379 | Event ingestion buffer |
| `prometheus` | `prom/prometheus:v2.51.0` | 9091 | Metrics collection |
| `grafana` | `grafana/grafana:11.0.0` | 3000 | Operational dashboards |
| `worker` | Custom (Dockerfile) | — | Event processing |
| `streamlit` | Custom (Dockerfile) | 8501 | Analytics dashboard |

**Dependency chain**: `redis` + `timescale` → `worker` → `streamlit` + `grafana` + `prometheus`

### 6.2 Dockerfile

The worker/streamlit container:
```dockerfile
FROM python:3.12-slim
COPY pyproject.toml + voicemon/ + alert_rules.yaml
RUN pip install ".[all]"
CMD ["python", "-m", "voicemon.workers.processor"]
```

### 6.3 Prometheus Configuration

`prometheus.yml` scrapes the worker's `/metrics` endpoint every 10 seconds. The worker's `PrometheusExporter` serves metrics on port 9090 via `prometheus_client.start_http_server()`.

### 6.4 Grafana Configuration

- `grafana-datasources.yml` — Provisions Prometheus and TimescaleDB as data sources.
- `provisioning.yaml` — Auto-loads the dashboard JSON from a file mount.
- `voicemon_dashboard.json` — Pre-built dashboard with all panels.

---

## 7. Configuration System

### 7.1 Environment Variables

| Variable | Default | Used By |
|----------|---------|---------|
| `VOICEMON_REDIS_URL` | `redis://localhost:6379/0` | Worker |
| `VOICEMON_TIMESCALE_DSN` | `postgresql://voicemon:voicemon@localhost:5432/voicemon` | Worker, Streamlit |
| `VOICEMON_SLACK_WEBHOOK` | (empty) | Worker |
| `VOICEMON_PAGERDUTY_KEY` | (empty) | Worker |

### 7.2 Alert Rules (`alert_rules.yaml`)

16 pre-configured rules covering all 4 observability layers:

**Layer 1 (Infra)**: `high_jitter` (>30ms), `high_packet_loss` (>3%), `low_mos_score` (<3.5)

**Layer 2 (Execution)**: `stt_latency_warning` (>500ms), `stt_latency_critical` (>1000ms), `stt_confidence_low` (<0.7), `stt_confidence_critical` (<0.5), `llm_ttft_warning` (>800ms), `llm_ttft_critical` (>2000ms), `tts_ttfb_warning` (>300ms), `tts_ttfb_critical` (>800ms)

**Layer 3 (UX)**: `e2e_latency_warning` (>1200ms), `e2e_latency_critical` (>1800ms, P0!), `high_interruption_rate` (>30%)

**Layer 4 (Outcome)**: `low_task_success` (<80%), `critical_task_failure` (<50%)

### 7.3 Industry-Standard Latency Thresholds

Built into `VoiceMonConfig.thresholds`:

| Metric | OK | Warning | Critical |
|--------|-----|---------|----------|
| E2E Latency | <800ms | <1200ms | >1800ms |
| STT Latency | <200ms | <350ms | — |
| LLM TTFT | <600ms | <1000ms | — |
| TTS TTFB | <150ms | <250ms | — |
| Jitter | <30ms | <50ms | — |
| Packet Loss | <0.5% | <1.0% | — |
| MOS Score | >4.0 | >3.5 | — |
| ASR Confidence | >0.90 | >0.85 | — |

---

## 8. Testing Strategy

### 8.1 Test Files

| File | Tests | What it validates |
|------|-------|-------------------|
| `test_models.py` | 10 tests | Data model creation, defaults, aggregation (TurnReport.e2e_latency, SessionReport computed properties) |
| `test_collector.py` | 9 tests | Session/turn lifecycle, recording all event types, multiple exporters, invalid session handling |
| `test_alerts.py` | 5 tests | Alert rule evaluation (>, <), defaults, engine initialization, threshold breach detection |
| `test_prometheus.py` | 3 tests | Exporter initialization, STT/LLM metric export, unknown event type handling |

### 8.2 Demo Simulator (`tests/demo_simulator.py`)

A simulation tool that generates realistic voice call telemetry with three quality profiles:

| Profile | STT Latency | LLM Latency | TTS Latency | Weight |
|---------|------------|-------------|-------------|--------|
| `good` | 150±30ms | 300±80ms | 120±25ms | 70% |
| `degraded` | 400±100ms | 800±200ms | 300±80ms | 20% |
| `bad` | 800±200ms | 2000±500ms | 600±150ms | 10% |

Simulates 2-8 turns per session with random STT providers, LLM models, TTS providers, transcripts, tool calls, interruptions, and outcomes. Supports console output (no infra needed) or Redis export (requires Docker stack).

### 8.3 Running Tests

```bash
pip install -e ".[dev]"   # Install dev dependencies
pytest                     # Run all tests
pytest -v                  # Verbose output
ruff check .               # Lint
mypy voicemon/             # Type check
```
