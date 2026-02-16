# VoiceMon - Visual Architecture Reference

This document provides visual representations of VoiceMon's architecture using ASCII diagrams.

## 1. System Overview - Bird's Eye View

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         VOICE AI APPLICATION LAYER                           │
│  ┌────────────┐        ┌────────────┐        ┌────────────┐                │
│  │  LiveKit   │        │  Pipecat   │        │    Vapi    │                │
│  │   Agent    │        │  Pipeline  │        │  Webhook   │                │
│  └──────┬─────┘        └──────┬─────┘        └──────┬─────┘                │
│         │                     │                      │                       │
│         └─────────────────────┼──────────────────────┘                       │
│                               │                                              │
│                      ┌────────▼─────────┐                                   │
│                      │  VoiceMon SDK    │                                   │
│                      │   (Collector)    │                                   │
│                      └────────┬─────────┘                                   │
└───────────────────────────────┼─────────────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐      ┌────────────────┐     ┌────────────────┐
│   Console     │      │ Redis Streams  │     │  Prometheus    │
│   (Debug)     │      │  (Ingestion)   │     │   (Metrics)    │
└───────────────┘      └────────┬───────┘     └────────┬───────┘
                                │                      │
                       ┌────────▼────────┐             │
                       │  Worker Pool    │             │
                       │  (Processing)   │             │
                       └────────┬────────┘             │
                                │                      │
                    ┌───────────┼───────────┐          │
                    │           │           │          │
                    ▼           ▼           ▼          ▼
            ┌──────────┐  ┌─────────┐  ┌────────┐  ┌────────┐
            │TimescaleDB│  │  Alert  │  │ Slack  │  │Grafana │
            │ (Storage) │  │ Engine  │  │PagerDuty│ │(Ops UI)│
            └─────┬─────┘  └─────────┘  └────────┘  └────────┘
                  │
                  ▼
          ┌────────────────┐
          │   Streamlit    │
          │ (Analytics UI) │
          └────────────────┘
```

## 2. Data Flow - Event Journey

### 2.1 Real-Time Event Flow (Hot Path)

```
User speaks → STT completes → LLM responds → TTS synthesizes
     │              │              │              │
     ▼              ▼              ▼              ▼
┌─────────────────────────────────────────────────────┐
│         VoiceMonCollector.record_*()                │
│  • record_stt()                                     │
│  • record_llm()                                     │
│  • record_tts()                                     │
│  • record_ux()                                      │
└──────────────────┬──────────────────────────────────┘
                   │ _emit(stream, event)
                   │
         ┌─────────┼─────────┬──────────┐
         │         │         │          │
         ▼         ▼         ▼          ▼
    ┌────────┐ ┌──────┐ ┌────────┐ ┌──────┐
    │Console │ │Redis │ │Prom    │ │OTel  │
    │Exporter│ │Export│ │Exporter│ │Spans │
    └────────┘ └──┬───┘ └───┬────┘ └──┬───┘
                  │         │          │
          ┌───────┘         │          └──────────┐
          │                 │                     │
          ▼                 ▼                     ▼
    ┌───────────┐     ┌──────────┐        ┌──────────┐
    │Redis Stream│    │Prometheus│        │  Jaeger  │
    │voicemon:stt│    │/metrics  │        │  (Trace) │
    └─────┬──────┘    └─────┬────┘        └──────────┘
          │                 │
          │                 └─────┐
          │                       │
          ▼                       ▼
    ┌───────────┐           ┌─────────┐
    │  Worker   │           │ Grafana │
    │ Processor │───────────│Dashboard│
    └─────┬─────┘           └─────────┘
          │
          └─────┬──────┬────────┐
                │      │        │
                ▼      ▼        ▼
           ┌─────┐ ┌────┐ ┌──────┐
           │TSDB │ │Alert│ │Stats │
           │Write│ │Check│ │ Agg  │
           └─────┘ └──┬─┘ └──────┘
                      │
                      ▼
                 ┌─────────┐
                 │  Slack  │
                 │PagerDuty│
                 └─────────┘
```

### 2.2 Query Path (Cold Path)

```
       User Query
           │
    ┌──────┴──────┐
    │             │
    ▼             ▼
┌─────────┐  ┌──────────┐
│ Grafana │  │Streamlit │
│  (Ops)  │  │(Analytics)│
└────┬────┘  └────┬─────┘
     │            │
     ├────────────┘
     │
┌────┴────────────────────────┐
│                             │
▼                             ▼
┌────────────────┐    ┌──────────────┐
│  Prometheus    │    │ TimescaleDB  │
│   (Metrics)    │    │ (Raw Events) │
│                │    │              │
│ • Histograms   │    │ • Sessions   │
│ • Counters     │    │ • Turns      │
│ • Gauges       │    │ • Events     │
└────────────────┘    └──────────────┘
```

## 3. VoiceMonCollector - Internal Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   VoiceMonCollector                          │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │          In-Memory State                           │    │
│  │                                                     │    │
│  │  _sessions: dict[session_id → VoiceSession]       │    │
│  │  _turns: dict[turn_id → Turn]                     │    │
│  │  _turn_reports: dict[turn_id → TurnReport]        │    │
│  │  _session_turns: dict[session_id → [turn_ids]]    │    │
│  │                                                     │    │
│  │  _lock: asyncio.Lock (thread-safety)              │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │          Public API                                │    │
│  │                                                     │    │
│  │  start_session() → VoiceSession                   │    │
│  │  end_session()   → VoiceSession                   │    │
│  │  start_turn()    → Turn                           │    │
│  │  end_turn()      → Turn                           │    │
│  │                                                     │    │
│  │  record_stt()    → STTEvent                       │    │
│  │  record_llm()    → LLMEvent                       │    │
│  │  record_tts()    → TTSEvent                       │    │
│  │  record_ux()     → UXEvent                        │    │
│  │  record_infra()  → InfraEvent                     │    │
│  │  record_outcome() → OutcomeEvent                  │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │          Exporter Pipeline                         │    │
│  │                                                     │    │
│  │  _emit(stream, event):                            │    │
│  │    for exporter in _exporters:                    │    │
│  │      await exporter.export(stream, event)         │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

## 4. Redis Streams - Consumer Group Pattern

```
┌────────────────────────────────────────────────────────────┐
│             Redis Stream: voicemon:stt                      │
│                                                             │
│  [msg-1] [msg-2] [msg-3] [msg-4] [msg-5] [msg-6] [msg-7]  │
│                                                             │
│  Consumer Group: voicemon-workers                          │
│  ├─ Last Delivered: msg-7                                  │
│  └─ Pending: msg-5, msg-6 (not yet ACKed)                 │
└─────────────────────────────────────────────────────────────┘
                         │
           ┌─────────────┼─────────────┐
           │             │             │
           ▼             ▼             ▼
    ┌───────────┐ ┌───────────┐ ┌───────────┐
    │ worker-1  │ │ worker-2  │ │ worker-3  │
    │ (pod 1)   │ │ (pod 2)   │ │ (pod 3)   │
    └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
          │             │             │
          │ XREADGROUP  │             │
          │ (block)     │             │
          │             │             │
          ▼             ▼             ▼
    Reads msg-1   Reads msg-2   Reads msg-3
    Processes     Processes     Processes
    ACKs msg-1    ACKs msg-2    ACKs msg-3
          │             │             │
          ▼             ▼             ▼
    ┌──────────────────────────────────────┐
    │        TimescaleDB                   │
    │  (writes are serialized by TSDB)     │
    └──────────────────────────────────────┘

If worker-2 crashes:
  msg-2 not ACKed → Redis redelivers to worker-1 or worker-3
  (At-least-once delivery)
```

## 5. Worker Processor - Internal Flow

```
┌──────────────────────────────────────────────────────────────┐
│                  MetricsProcessor                             │
│                                                               │
│  Start:                                                       │
│    ├─ Connect Redis                                          │
│    ├─ Connect TimescaleDB                                    │
│    ├─ Initialize AlertEngine                                 │
│    └─ Spawn 8 consumer loops (one per event type)           │
│                                                               │
│  Consumer Loop (for each event type):                        │
│    while running:                                            │
│      ├─ XREADGROUP(stream, consumer_name, count=100)        │
│      ├─ For each message:                                    │
│      │   ├─ _process_event()                                │
│      │   │   ├─ _persist() → TimescaleDB                    │
│      │   │   ├─ _update_windows() → Rolling stats           │
│      │   │   └─ _check_thresholds() → Alert if breach       │
│      │   └─ Add to ack_ids                                  │
│      └─ XACK(stream, *ack_ids)                              │
└──────────────────────────────────────────────────────────────┘

Rolling Windows (in-memory):
  _windows = {
    "stt_latency":  deque([120, 105, 98, 110, ...], maxlen=100),
    "llm_ttft":     deque([450, 380, 420, ...], maxlen=100),
    "tts_ttfb":     deque([150, 130, 145, ...], maxlen=100),
    ...
  }

Alert Cooldowns (anti-spam):
  _alert_cooldowns = {
    "stt_latency_critical:agent-1": 1709876543.21,
    "e2e_latency_warning:agent-2": 1709876600.00,
    ...
  }
```

## 6. Data Model Hierarchy - 4-Layer Framework

```
┌──────────────────────────────────────────────────────────┐
│               SESSION (envelope)                          │
│  session_id, agent_id, framework, started_at, ended_at  │
│  status, duration_ms, total_turns, total_cost_usd       │
└────────────────────┬─────────────────────────────────────┘
                     │
         ┌───────────┴───────────┬────────────┬────────────┐
         │                       │            │            │
         ▼                       ▼            ▼            ▼
    ┌────────┐            ┌────────┐    ┌────────┐  ┌─────────┐
    │ TURN 1 │            │ TURN 2 │    │ TURN 3 │  │ OUTCOME │
    │ (user) │            │ (agent)│    │ (user) │  │  EVENT  │
    └───┬────┘            └───┬────┘    └───┬────┘  └─────────┘
        │                     │             │
        └─────────┬───────────┘             │
                  │                         │
    ┌─────────────┴─────────────────────────┘
    │
    ├──────┬──────┬──────┬──────┬──────┐
    │      │      │      │      │      │
    ▼      ▼      ▼      ▼      ▼      ▼

LAYER 1     LAYER 2              LAYER 3     LAYER 4
InfraEvent  STTEvent             UXEvent     OutcomeEvent
├─jitter    ├─transcript         ├─e2e_lat   ├─task_completed
├─pkt_loss  ├─confidence         ├─ttfw_ms   ├─task_name
├─mos       ├─latency_ms         ├─interrupts├─escalated
└─rtt       └─provider            └─sentiment └─cost_usd

            LLMEvent
            ├─model
            ├─ttft_ms
            ├─tokens
            └─cost_usd

            TTSEvent
            ├─provider
            ├─ttfb_ms
            ├─voice_id
            └─audio_dur
```

## 7. Alert Engine - Rule Evaluation Flow

```
                Event Received
                      │
                      ▼
            ┌──────────────────┐
            │  _check_thresh() │
            └────────┬─────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
  Threshold Check         Anomaly Detection
  (Static Rules)          (Z-score)
        │                         │
        │  e2e_latency > 1800?   │  |Z| > 3.0?
        │  stt_conf < 0.7?       │
        │                         │
        └────────────┬────────────┘
                     │
                     ▼  Yes
            ┌─────────────────┐
            │  _fire_alert()  │
            └────────┬────────┘
                     │
                     ▼
            Cooldown Check
            ┌──────────────────┐
            │ Last fired when? │
            │ Now - Last > 5min?│
            └────────┬─────────┘
                     │ Yes
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
  ┌──────────┐            ┌─────────────┐
  │TimescaleDB│           │  Notify     │
  │ (persist  │           │             │
  │  alert)   │           ├──Slack      │
  └───────────┘           ├──PagerDuty  │
                          └─────────────┘

Alert Routing by Severity:
  info     → Slack #voicemon-info
  warning  → Slack #voicemon-alerts
  critical → Slack #voicemon-alerts + @oncall mention
  p0       → Slack + PagerDuty incident
```

## 8. TimescaleDB - Schema Design

```
┌────────────────────────────────────────────────────────┐
│                    SESSIONS TABLE                       │
│  (parent table)                                        │
│                                                        │
│  session_id (PK)  ← unique identifier                 │
│  agent_id         ← which agent                       │
│  framework        ← livekit/pipecat/vapi              │
│  started_at       ← timestamp (hypertable dimension)  │
│  ended_at                                             │
│  status           ← active/completed/failed           │
│  duration_ms                                          │
│  total_turns                                          │
│  total_cost_usd                                       │
│  metadata         ← JSONB (flexible)                  │
│  tags             ← TEXT[] (filtering)                │
└────────────┬───────────────────────────────────────────┘
             │ 1:N
             ▼
┌────────────────────────────────────────────────────────┐
│                     TURNS TABLE                         │
│  (child table)                                         │
│                                                        │
│  turn_id (PK)                                          │
│  session_id (FK) → sessions.session_id                │
│  turn_number                                           │
│  speaker          ← user/agent/system                 │
│  started_at       ← timestamp                         │
│  ended_at                                             │
│  duration_ms                                          │
│  was_interrupted  ← boolean                           │
│  transcript       ← TEXT                              │
│  metadata         ← JSONB                             │
└────────────┬───────────────────────────────────────────┘
             │ 1:N
             ├─────────────┬──────────┬──────────┬──────┐
             │             │          │          │      │
             ▼             ▼          ▼          ▼      ▼
┌───────────────┐  ┌───────────┐  ┌────────┐  ┌────┐  ┌──────┐
│  stt_events   │  │llm_events │  │tts_evts│  │ ux │  │infra │
│               │  │           │  │        │  │evts│  │evts  │
│ event_id (PK) │  │event_id   │  │event_id│  │    │  │      │
│ session_id(FK)│  │session_id │  │sess_id │  │    │  │      │
│ turn_id (FK)  │  │turn_id    │  │turn_id │  │    │  │      │
│ timestamp     │  │timestamp  │  │timestamp│  │    │  │      │
│ transcript    │  │model      │  │provider │  │e2e │  │jitter│
│ confidence    │  │ttft_ms    │  │ttfb_ms  │  │lat │  │mos   │
│ latency_ms    │  │tokens     │  │voice_id │  │int │  │rtt   │
│ provider      │  │cost_usd   │  │duration │  │    │  │      │
└───────────────┘  └───────────┘  └────────┘  └────┘  └──────┘

                         ┌──────────────┐
                         │outcome_events│
                         │              │
                         │ event_id (PK)│
                         │ session_id(FK)│
                         │ timestamp    │
                         │task_completed│
                         │ escalated    │
                         │ cost_usd     │
                         └──────────────┘

Hypertables (TimescaleDB):
  - Partitioned by time (auto-managed)
  - Fast range queries (WHERE timestamp > NOW() - INTERVAL '1 hour')
  - Compression (old data)
  - Retention policies (auto-delete after 30 days)
```

## 9. Prometheus Metrics - Instrumentation

```
┌──────────────────────────────────────────────────────────┐
│              Prometheus Exporter                          │
│                                                           │
│  HISTOGRAMS (latency distributions):                     │
│    voicemon_stt_latency_ms{provider,model}              │
│    voicemon_llm_ttft_ms{provider,model}                 │
│    voicemon_tts_ttfb_ms{provider}                       │
│    voicemon_e2e_latency_ms{agent_id}                    │
│    voicemon_stt_confidence{provider}                    │
│                                                           │
│  COUNTERS (totals):                                      │
│    voicemon_sessions_total{agent_id,framework}          │
│    voicemon_turns_total{agent_id,speaker}               │
│    voicemon_interruptions_total{agent_id}               │
│    voicemon_llm_tokens_total{provider,direction}        │
│    voicemon_errors_total{agent_id,component}            │
│                                                           │
│  GAUGES (current values):                                │
│    voicemon_active_sessions{agent_id}                   │
│    voicemon_stt_confidence_current{agent_id}            │
│    voicemon_mos_score{agent_id}                         │
└──────────────────────────────────────────────────────────┘
                         │
                         │ HTTP /metrics
                         ▼
┌──────────────────────────────────────────────────────────┐
│                    Prometheus Server                      │
│                                                           │
│  Scrapes every 15s:                                      │
│    GET http://voicemon:9090/metrics                     │
│                                                           │
│  Stores time-series:                                     │
│    voicemon_stt_latency_ms{provider="deepgram",         │
│                             model="nova-2"}              │
│      [t=1709876540] bucket{le="100"} = 5                │
│      [t=1709876540] bucket{le="200"} = 12               │
│      [t=1709876540] bucket{le="500"} = 45               │
│                                                           │
│  PromQL Queries:                                         │
│    histogram_quantile(0.95, voicemon_stt_latency_ms)    │
│    rate(voicemon_sessions_total[5m])                    │
│    avg(voicemon_mos_score) by (agent_id)                │
└──────────────────────────────────────────────────────────┘
```

## 10. OpenTelemetry - Span Hierarchy

```
┌──────────────────────────────────────────────────────────┐
│  Span: voice_session                                      │
│    trace_id: a1b2c3d4e5f6                                │
│    span_id:  1111                                        │
│    attributes:                                           │
│      voice.session.id = "sess-abc123"                   │
│      voice.agent.id = "customer-support-v2"             │
│      voice.framework = "livekit"                        │
│    duration: 120s                                        │
│                                                           │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Span: voice_turn                                   │ │
│  │    parent_span_id: 1111                            │ │
│  │    span_id: 2222                                   │ │
│  │    attributes:                                      │ │
│  │      voice.turn.number = 1                         │ │
│  │      voice.turn.speaker = "user"                   │ │
│  │    duration: 5.2s                                  │ │
│  │                                                     │ │
│  │  ┌──────────────────────────────────────────────┐ │ │
│  │  │  Span: stt_recognition                        │ │ │
│  │  │    parent_span_id: 2222                      │ │ │
│  │  │    span_id: 3333                             │ │ │
│  │  │    attributes:                                │ │ │
│  │  │      voice.stt.provider = "deepgram"         │ │ │
│  │  │      voice.stt.confidence = 0.92             │ │ │
│  │  │      voice.stt.latency_ms = 150              │ │ │
│  │  │    duration: 150ms                           │ │ │
│  │  └──────────────────────────────────────────────┘ │ │
│  │                                                     │ │
│  │  ┌──────────────────────────────────────────────┐ │ │
│  │  │  Span: llm_inference                         │ │ │
│  │  │    parent_span_id: 2222                      │ │ │
│  │  │    span_id: 4444                             │ │ │
│  │  │    attributes:                                │ │ │
│  │  │      gen_ai.provider.name = "openai"         │ │ │
│  │  │      gen_ai.request.model = "gpt-4o"         │ │ │
│  │  │      voice.llm.ttft_ms = 450                 │ │ │
│  │  │      gen_ai.usage.input_tokens = 120         │ │ │
│  │  │      gen_ai.usage.output_tokens = 80         │ │ │
│  │  │    duration: 1200ms                          │ │ │
│  │  │                                               │ │ │
│  │  │  ┌────────────────────────────────────────┐ │ │ │
│  │  │  │  Span: tool_call                        │ │ │
│  │  │  │    parent_span_id: 4444                │ │ │
│  │  │  │    span_id: 5555                       │ │ │
│  │  │  │    attributes:                          │ │ │
│  │  │  │      voice.tool.function_name =        │ │ │
│  │  │  │        "lookup_account"                │ │ │
│  │  │  │    duration: 250ms                     │ │ │
│  │  │  └────────────────────────────────────────┘ │ │ │
│  │  └──────────────────────────────────────────────┘ │ │
│  │                                                     │ │
│  │  ┌──────────────────────────────────────────────┐ │ │
│  │  │  Span: tts_synthesis                         │ │ │
│  │  │    parent_span_id: 2222                      │ │ │
│  │  │    span_id: 6666                             │ │ │
│  │  │    attributes:                                │ │ │
│  │  │      voice.tts.provider = "elevenlabs"       │ │ │
│  │  │      voice.tts.ttfb_ms = 120                 │ │ │
│  │  │    duration: 800ms                           │ │ │
│  │  └──────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘

Visualized in Jaeger:
  [=========== voice_session (120s) ===========]
    [== turn ==]
      [stt]
      [==== llm ====]
        [tool]
      [== tts ==]
```

## 11. LiveKit Integration - Hook Points

```
┌─────────────────────────────────────────────────────────┐
│              LiveKit Agent                               │
│                                                          │
│  AgentSession Events:                                   │
│    • metrics_collected                                  │
│    • user_state_changed                                 │
│    • agent_state_changed                                │
│    • user_input_transcribed                            │
│    • agent_speech_interrupted                          │
│    • function_tools_executed                           │
│    • close                                              │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ instrument_livekit(session, collector)
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│           VoiceMon Event Handlers                        │
│                                                          │
│  @on("metrics_collected"):                              │
│    ├─ Extract STT metrics → collector.record_stt()     │
│    ├─ Extract LLM metrics → collector.record_llm()     │
│    └─ Extract TTS metrics → collector.record_tts()     │
│                                                          │
│  @on("user_state_changed"):                             │
│    ├─ "speaking" → collector.start_turn(speaker=user)  │
│    └─ "listening" → (turn continues)                   │
│                                                          │
│  @on("agent_state_changed"):                            │
│    ├─ "speaking" → collector.start_turn(speaker=agent) │
│    └─ "listening" → collector.end_turn()               │
│                                                          │
│  @on("agent_speech_interrupted"):                       │
│    └─ collector.record_ux(interruption_count=1)        │
│                                                          │
│  @on("function_tools_executed"):                        │
│    └─ collector.record_llm(provider="tool_call")       │
│                                                          │
│  @on("close"):                                          │
│    └─ collector.end_session()                          │
└─────────────────────────────────────────────────────────┘
```

## 12. Failure Recovery - At-Least-Once Delivery

```
┌──────────────────────────────────────────────────────────┐
│                 Failure Scenario                          │
└──────────────────────────────────────────────────────────┘

Worker-1 reads 100 messages from Redis Stream
    ├─ msg-1  [processed] ✓
    ├─ msg-2  [processed] ✓
    │  ...
    ├─ msg-49 [processed] ✓
    ├─ msg-50 [processing...] 
    └─ CRASH! (OOM, segfault, etc.)

Redis Stream:
  - msg-1 to msg-49: PENDING (not ACKed)
  - msg-50 to msg-100: PENDING (not ACKed)

After timeout (10 minutes):
  Redis: "These messages are still PENDING, reassign them"

Worker-2 (or restarted Worker-1):
  XREADGROUP → Receives msg-1 to msg-100 again

  For each message:
    ├─ _process_event()
    │   └─ TimescaleDB: INSERT ... ON CONFLICT DO NOTHING
    │      (Idempotent! msg-1 to msg-49 already exist, skip)
    └─ ACK message

Result:
  ✓ All 100 messages processed
  ✓ No data loss
  ✓ At-most duplicate writes (handled by idempotency)
```

## 13. Horizontal Scaling - Adding Workers

```
Before (1 worker):
  Redis Queue: [||||||||||||||||||||||||||||||||] 10,000 msgs
  Worker-1: Processing 100 msgs/sec
  Throughput: 100 msgs/sec
  Lag: 100 seconds

After (3 workers):
  Redis Queue: [||||||||||||||||||||||||||||||||] 10,000 msgs
  Worker-1: ▓▓▓▓▓▓▓▓▓▓ (33%)
  Worker-2: ▓▓▓▓▓▓▓▓▓▓ (33%)
  Worker-3: ▓▓▓▓▓▓▓▓▓▓ (33%)
  Throughput: 300 msgs/sec
  Lag: 33 seconds

How it works:
  ┌─────────────────────────────────────────┐
  │  Redis Consumer Group Load Balancing    │
  │                                         │
  │  XREADGROUP distributes messages:       │
  │    msg-1 → worker-1                    │
  │    msg-2 → worker-2                    │
  │    msg-3 → worker-3                    │
  │    msg-4 → worker-1                    │
  │    msg-5 → worker-2                    │
  │    ...                                  │
  │                                         │
  │  Each worker processes independently    │
  │  No coordination needed                │
  └─────────────────────────────────────────┘
```

## 14. Alert Cooldown - Preventing Spam

```
Timeline:
  T+0s:   e2e_latency = 2000ms (> 1800ms threshold)
          → Fire alert "e2e_latency_critical"
          → Slack notification sent
          → PagerDuty incident created
          → Cooldown set: 2 minutes

  T+10s:  e2e_latency = 2100ms (still > threshold)
          → Check cooldown: Now - T+0s = 10s < 120s
          → SKIP (still in cooldown, don't spam)

  T+60s:  e2e_latency = 1900ms (still > threshold)
          → Check cooldown: Now - T+0s = 60s < 120s
          → SKIP

  T+130s: e2e_latency = 2000ms (still > threshold)
          → Check cooldown: Now - T+0s = 130s > 120s
          → Fire alert again (cooldown expired)
          → Slack notification sent

Cooldown State (in-memory):
  _alert_cooldowns = {
    "e2e_latency_critical:agent-1": 1709876543.21,  # T+0s
    "stt_confidence_low:agent-2":   1709876600.00,
    ...
  }

Benefits:
  ✓ Prevents alert fatigue
  ✓ Reduces Slack/PagerDuty noise
  ✓ Still notifies if issue persists (after cooldown)
```

## 15. Anomaly Detection - Z-Score Visualization

```
Rolling Window (last 100 values):
  stt_latency_ms: [100, 105, 98, 102, 110, 95, 103, ...]

Normal Distribution:
           │                     
           │        ▄▄▄▄▄        
           │     ▄▄▀     ▀▄▄     
           │   ▄▀           ▀▄   
           │ ▄▀               ▀▄ 
  ─────────┼──────────────────────────
           85    100   115   500

  Mean (μ) = 101.7
  Stdev (σ) = 5.5

  Z-score = (X - μ) / σ

  X = 105:  Z = (105 - 101.7) / 5.5 = 0.6  (normal)
  X = 115:  Z = (115 - 101.7) / 5.5 = 2.4  (borderline)
  X = 500:  Z = (500 - 101.7) / 5.5 = 72.4 (ANOMALY!)

Threshold: |Z| > 3.0 = anomaly

Example Alert:
  "Anomaly detected: stt_latency=500ms 
   (z-score=72.4, mean=101.7ms, stdev=5.5ms)"
```

---

## Summary - Key Architectural Patterns

1. **Fan-out Pattern**: Collector → Multiple Exporters
2. **Consumer Group**: Redis Streams → Multiple Workers
3. **At-least-once Delivery**: Redis pending + ACK
4. **Idempotent Operations**: INSERT ON CONFLICT DO NOTHING
5. **Async-first**: Non-blocking I/O throughout
6. **Circuit Breaker**: Exporter failures don't break collector
7. **Rolling Windows**: Fixed-size deques for real-time stats
8. **Z-score Anomaly Detection**: Statistical outlier detection
9. **Alert Cooldown**: Rate limiting to prevent spam
10. **Graceful Degradation**: Missing deps don't crash system
11. **Horizontal Scaling**: Add workers to handle more load
12. **Hypertables**: TimescaleDB time-series partitioning
13. **Voice-aware Spans**: OpenTelemetry with custom attributes
14. **4-Layer Framework**: Infra → Execution → UX → Outcome

---

This visual guide complements the detailed `ARCHITECTURE_GUIDE.md` by providing
ASCII diagrams that illustrate the system's architecture, data flows, and
key design patterns.
