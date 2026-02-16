# VoiceMon - Quick Reference Guide

A concise reference for developers working with VoiceMon.

## Quick Start - 3 Lines of Code

```python
from voicemon import VoiceMonCollector
from voicemon.integrations.livekit import instrument_livekit

collector = VoiceMonCollector()
await collector.start()
instrument_livekit(agent_session, collector, agent_id="my-agent")
```

## Core Concepts Cheat Sheet

### 4-Layer Observability Framework

| Layer | Measures | Key Metrics | Why It Matters |
|-------|----------|-------------|----------------|
| **L1: Infrastructure** | Network & codec | Jitter, packet loss, MOS | Poor network = choppy audio |
| **L2: Execution** | Pipeline components | STT latency, LLM TTFT, TTS TTFB | Slow components = user waits |
| **L3: User Experience** | Perceived quality | E2E latency, interruptions | What users actually feel |
| **L4: Outcome** | Business results | Task success, cost, CSAT | Did it work? At what cost? |

### Data Model Hierarchy

```
VoiceSession (session_id)
  └─ Turn[] (turn_id)
      ├─ STTEvent (transcript, confidence, latency)
      ├─ LLMEvent (model, ttft, tokens)
      ├─ TTSEvent (provider, ttfb, voice_id)
      ├─ UXEvent (e2e_latency, interruptions)
      ├─ InfraEvent (jitter, packet_loss, mos)
      └─ OutcomeEvent (task_completed, cost)
```

## API Reference

### Session Management

```python
# Start session
session = collector.start_session(
    agent_id="customer-support-v2",
    framework="livekit",
    metadata={"env": "production"}
)

# End session
await collector.end_session(
    session.session_id,
    status=SessionStatus.COMPLETED,
    task_completed=True
)
```

### Turn Management

```python
# Start user turn
turn = collector.start_turn(
    session_id=session.session_id,
    speaker=Speaker.USER
)

# End turn
await collector.end_turn(
    turn.turn_id,
    was_interrupted=False,
    transcript="I want to check my balance"
)
```

### Recording Events

#### Layer 2: Execution Events

```python
# STT (Speech-to-Text)
await collector.record_stt(
    session_id=session_id,
    turn_id=turn_id,
    transcript="Hello, how can I help you?",
    confidence=0.92,           # 0.0-1.0 (> 0.85 is good)
    latency_ms=150,            # < 300ms is good
    provider="deepgram",
    model="nova-2"
)

# LLM (Language Model)
await collector.record_llm(
    session_id=session_id,
    turn_id=turn_id,
    model="gpt-4o",
    provider="openai",
    ttft_ms=450,               # Time to first token (< 600ms is good)
    prompt_tokens=120,
    completion_tokens=80,
    cost_usd=0.003             # Track spend
)

# TTS (Text-to-Speech)
await collector.record_tts(
    session_id=session_id,
    turn_id=turn_id,
    provider="elevenlabs",
    voice_id="EXAVITQu4vr4xnSDS",
    ttfb_ms=120,               # Time to first byte (< 200ms is good)
    characters_count=45,
    cost_usd=0.001
)
```

#### Layer 1: Infrastructure

```python
await collector.record_infra(
    session_id=session_id,
    jitter_ms=15,              # < 30ms is good
    packet_loss_pct=0.5,       # < 1% is good
    mos_score=4.2,             # Mean Opinion Score 1-5 (> 4.0 is good)
    rtt_ms=50                  # Round-trip time
)
```

#### Layer 3: User Experience

```python
await collector.record_ux(
    session_id=session_id,
    turn_id=turn_id,
    e2e_latency_ms=720,        # End-to-end (< 800ms is excellent)
    interruption_count=1,
    sentiment="positive"
)
```

#### Layer 4: Outcome

```python
await collector.record_outcome(
    session_id=session_id,
    task_completed=True,
    task_name="balance_inquiry",
    turns_to_complete=3,
    escalated=False,
    cost_usd=0.015
)
```

## Exporters

### Console Exporter (Debug)

```python
from voicemon.exporters.base import ConsoleExporter

collector = VoiceMonCollector()
collector.add_exporter(ConsoleExporter(verbose=True))
```

### Redis Exporter (Production)

```python
from voicemon.exporters.redis import RedisStreamsExporter

redis_exp = RedisStreamsExporter(redis_url="redis://localhost:6379/0")
await redis_exp.connect()
collector.add_exporter(redis_exp)
```

### Prometheus Exporter (Metrics)

```python
from voicemon.exporters.prometheus import PrometheusExporter

prom_exp = PrometheusExporter(port=9090)
prom_exp.start_server()  # Starts /metrics endpoint
collector.add_exporter(prom_exp)
```

## Integrations

### LiveKit

```python
from livekit.agents import AgentSession
from voicemon.integrations.livekit import instrument_livekit

session = AgentSession(...)
session_id = instrument_livekit(
    session, 
    collector, 
    agent_id="livekit-support-bot"
)
# VoiceMon automatically captures metrics from LiveKit events
```

### Pipecat

```python
from pipecat.pipeline.pipeline import Pipeline
from voicemon.integrations.pipecat import VoiceMonPipecatObserver

observer = VoiceMonPipecatObserver(collector, agent_id="pipecat-agent")
pipeline = Pipeline([...], observers=[observer])
```

### Vapi (Webhooks)

```python
from fastapi import FastAPI
from voicemon.integrations.vapi import create_vapi_router

app = FastAPI()
router = create_vapi_router(collector, webhook_secret="your-secret")
app.include_router(router)
```

### Vapi (Polling)

```python
from voicemon.integrations.vapi import VapiClient

client = VapiClient(api_key="your-key", collector=collector)
session_ids = await client.poll_recent_calls(minutes=5)
```

## Configuration

### Environment Variables

```bash
# Redis
VOICEMON_REDIS_URL=redis://localhost:6379/0

# TimescaleDB
VOICEMON_TIMESCALE_DSN=postgresql://voicemon:voicemon@localhost:5432/voicemon

# Slack
VOICEMON_SLACK_WEBHOOK=https://hooks.slack.com/services/...

# PagerDuty
VOICEMON_PAGERDUTY_KEY=your-routing-key
```

### Config Object

```python
from voicemon import VoiceMonConfig

config = VoiceMonConfig(
    redis={"url": "redis://localhost:6379/0"},
    timescale={"dsn": "postgresql://..."},
    prometheus={"enabled": True, "port": 9090},
    alerts={
        "rules_file": "alert_rules.yaml",
        "slack": {
            "webhook_url": "https://hooks.slack.com/...",
            "cooldown_seconds": 900
        }
    }
)

collector = VoiceMonCollector(config)
```

## Industry-Standard Thresholds

### Latency (Lower is Better)

| Metric | Excellent | Good | Warning | Critical |
|--------|-----------|------|---------|----------|
| **E2E Latency** | < 600ms | < 800ms | < 1200ms | > 1800ms |
| **STT Latency** | < 150ms | < 300ms | < 500ms | > 1000ms |
| **LLM TTFT** | < 400ms | < 600ms | < 800ms | > 2000ms |
| **TTS TTFB** | < 100ms | < 200ms | < 300ms | > 800ms |

### Quality (Higher is Better)

| Metric | Excellent | Good | Warning | Critical |
|--------|-----------|------|---------|----------|
| **STT Confidence** | > 0.92 | > 0.85 | > 0.7 | < 0.5 |
| **MOS Score** | > 4.0 | > 3.5 | > 3.0 | < 2.5 |

### Network (Lower is Better)

| Metric | Excellent | Good | Warning | Critical |
|--------|-----------|------|---------|----------|
| **Jitter** | < 15ms | < 30ms | < 50ms | > 100ms |
| **Packet Loss** | < 0.5% | < 1% | < 3% | > 5% |

## Common Queries

### PromQL (Prometheus)

```promql
# p95 STT latency for Deepgram
histogram_quantile(0.95, 
  voicemon_stt_latency_ms{provider="deepgram"}
)

# Session rate per minute
rate(voicemon_sessions_total[1m])

# Average MOS score by agent
avg(voicemon_mos_score) by (agent_id)

# Error rate
rate(voicemon_errors_total[5m]) > 0
```

### SQL (TimescaleDB)

```sql
-- Recent sessions with high latency
SELECT s.session_id, s.agent_id, AVG(ux.e2e_latency_ms) as avg_latency
FROM sessions s
JOIN ux_events ux ON s.session_id = ux.session_id
WHERE s.started_at > NOW() - INTERVAL '1 hour'
GROUP BY s.session_id, s.agent_id
HAVING AVG(ux.e2e_latency_ms) > 1200
ORDER BY avg_latency DESC;

-- STT confidence trend (last 24 hours)
SELECT 
  time_bucket('1 hour', timestamp) as hour,
  AVG(confidence) as avg_confidence,
  COUNT(*) as event_count
FROM stt_events
WHERE timestamp > NOW() - INTERVAL '24 hours'
  AND is_final = TRUE
GROUP BY hour
ORDER BY hour;

-- Task success rate by agent
SELECT 
  s.agent_id,
  COUNT(*) as total_sessions,
  SUM(CASE WHEN o.task_completed THEN 1 ELSE 0 END) as successful,
  ROUND(100.0 * SUM(CASE WHEN o.task_completed THEN 1 ELSE 0 END) / COUNT(*), 2) as success_rate
FROM sessions s
JOIN outcome_events o ON s.session_id = o.session_id
WHERE s.started_at > NOW() - INTERVAL '7 days'
GROUP BY s.agent_id
ORDER BY success_rate DESC;

-- Cost analysis
SELECT 
  agent_id,
  COUNT(*) as sessions,
  SUM(total_cost_usd) as total_cost,
  AVG(total_cost_usd) as avg_cost_per_session,
  SUM(total_turns) as total_turns
FROM sessions
WHERE started_at > NOW() - INTERVAL '1 day'
GROUP BY agent_id;
```

## Alert Rules Example

```yaml
# alert_rules.yaml
rules:
  - name: e2e_latency_critical
    description: "Users experiencing unacceptable lag"
    metric: e2e_latency_ms
    operator: ">"
    threshold: 1800
    severity: p0
    cooldown_minutes: 2
    notify: [slack, pagerduty]

  - name: stt_confidence_low
    description: "STT accuracy degraded"
    metric: stt_confidence
    operator: "<"
    threshold: 0.7
    severity: warning
    cooldown_minutes: 10
    notify: [slack]
```

## Docker Commands

```bash
# Start full stack
docker compose up -d

# Initialize database
docker compose exec timescale psql -U voicemon -d voicemon -f /schema/schema.sql

# View logs
docker compose logs -f worker
docker compose logs -f streamlit

# Scale workers
docker compose up -d --scale worker=3

# Stop all
docker compose down

# Reset everything
docker compose down -v  # WARNING: Deletes data
```

## Demo Simulator

Generate realistic test data:

```bash
# Console output (no infrastructure needed)
python -m tests.demo_simulator --sessions 50

# Export to Redis (requires docker compose)
python -m tests.demo_simulator --sessions 100 --redis

# Custom parameters
python -m tests.demo_simulator \
  --sessions 200 \
  --agent-id my-test-agent \
  --concurrency 10 \
  --redis
```

## Debugging Tips

### Check if collector is recording

```python
collector = VoiceMonCollector()
collector.add_exporter(ConsoleExporter(verbose=True))
await collector.start()

# Should see output:
# [voicemon:stt] {"transcript": "hello", ...}
```

### Check Redis queue depth

```bash
docker compose exec redis redis-cli

# List streams
KEYS voicemon:*

# Check stream length
XLEN voicemon:stt

# View consumer groups
XINFO GROUPS voicemon:stt

# Peek at messages
XRANGE voicemon:stt - + COUNT 5
```

### Check TimescaleDB data

```bash
docker compose exec timescale psql -U voicemon -d voicemon

# Count events
SELECT 'sessions' as table, COUNT(*) FROM sessions
UNION ALL
SELECT 'turns', COUNT(*) FROM turns
UNION ALL
SELECT 'stt_events', COUNT(*) FROM stt_events;

# Recent sessions
SELECT session_id, agent_id, started_at, status 
FROM sessions 
ORDER BY started_at DESC 
LIMIT 10;
```

### Check Prometheus metrics

```bash
# Scrape /metrics endpoint
curl http://localhost:9090/metrics | grep voicemon

# Should see:
# voicemon_sessions_total{agent_id="my-agent"} 42
# voicemon_stt_latency_ms_bucket{le="200"} 150
```

## Performance Benchmarks

| Metric | Value | Notes |
|--------|-------|-------|
| Collector overhead | < 10ms | In-memory, async |
| Throughput per worker | ~10K events/sec | Batch processing |
| Storage per event | ~1KB | Compressed in TSDB |
| Query latency (p95) | < 100ms | Hypertables |
| Redis memory | ~256MB | For 100K messages |

## Common Patterns

### Track cost per session

```python
# During session
llm_cost = 0.0
tts_cost = 0.0

# After each LLM call
await collector.record_llm(..., cost_usd=0.003)
llm_cost += 0.003

# After each TTS call
await collector.record_tts(..., cost_usd=0.001)
tts_cost += 0.001

# End session
await collector.end_session(
    session_id,
    cost_usd=llm_cost + tts_cost
)
```

### Detect high-latency turns

```python
turn = collector.start_turn(session_id, speaker="user")
start = time.time()

# ... STT, LLM, TTS ...

duration = (time.time() - start) * 1000  # ms

if duration > 2000:  # > 2s
    await collector.record_ux(
        session_id=session_id,
        turn_id=turn.turn_id,
        e2e_latency_ms=duration,
        metadata={"high_latency_detected": True}
    )
```

### A/B test STT providers

```python
# Randomly select provider
import random
provider = random.choice(["deepgram", "google", "azure"])

await collector.record_stt(
    session_id=session_id,
    provider=provider,  # Track which provider
    latency_ms=latency,
    confidence=confidence,
    metadata={"experiment": "stt-provider-test"}
)

# Query in TimescaleDB:
# SELECT provider, AVG(latency_ms), AVG(confidence)
# FROM stt_events
# WHERE metadata->>'experiment' = 'stt-provider-test'
# GROUP BY provider;
```

### Monitor fallback triggers

```python
if stt_confidence < 0.5:
    # Fallback to re-prompt user
    await collector.record_ux(
        session_id=session_id,
        fallback_triggered=True,
        metadata={"fallback_reason": "low_stt_confidence"}
    )
```

## Troubleshooting

### "Redis connection refused"

```bash
# Check if Redis is running
docker compose ps redis

# Check logs
docker compose logs redis

# Restart
docker compose restart redis
```

### "Worker not processing events"

```bash
# Check worker logs
docker compose logs worker

# Check Redis queue depth (should decrease over time)
docker compose exec redis redis-cli XLEN voicemon:stt

# Check worker health
docker compose ps worker
```

### "Prometheus metrics not showing"

```python
# Ensure PrometheusExporter is initialized
from voicemon.exporters.prometheus import PrometheusExporter

prom = PrometheusExporter(port=9090)
prom.start_server()  # Must call this!
collector.add_exporter(prom)
```

### "Alerts not firing"

```bash
# Check alert rules are loaded
docker compose logs worker | grep "Loaded.*rules"

# Check alert engine initialized
docker compose logs worker | grep "AlertEngine"

# Check Slack webhook is configured
echo $VOICEMON_SLACK_WEBHOOK
```

## Production Checklist

- [ ] Configure Redis maxmemory (prevent OOM)
- [ ] Set TimescaleDB retention policies (auto-delete old data)
- [ ] Enable Prometheus remote write (long-term storage)
- [ ] Set up Grafana alerting (email, Slack, PagerDuty)
- [ ] Configure alert cooldowns (prevent spam)
- [ ] Scale workers based on Redis queue depth
- [ ] Monitor worker health (liveness/readiness probes)
- [ ] Backup TimescaleDB (pg_dump or continuous archiving)
- [ ] Rate limit alerts (max X per hour)
- [ ] Set up log aggregation (CloudWatch, Loki)
- [ ] Configure TLS for Redis/TimescaleDB connections
- [ ] Set resource limits (CPU/memory) for containers
- [ ] Enable authentication for all services
- [ ] Set up monitoring for monitoring (meta-monitoring)

## Further Reading

- **Full Architecture**: See `ARCHITECTURE_GUIDE.md` for detailed explanations
- **Visual Diagrams**: See `VISUAL_ARCHITECTURE.md` for ASCII diagrams
- **4-Layer Framework**: [Hamming AI Blog Post](https://www.hamming.ai/blog/voice-agent-observability)
- **OpenTelemetry Spec**: https://opentelemetry.io/docs/
- **TimescaleDB Docs**: https://docs.timescale.com/
- **Prometheus Best Practices**: https://prometheus.io/docs/practices/

---

**Quick Links:**
- GitHub: https://github.com/jaiswal-naman/voicemon
- Issues: https://github.com/jaiswal-naman/voicemon/issues
- License: Apache-2.0
