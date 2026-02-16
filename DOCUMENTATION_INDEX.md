# Documentation Index

Welcome to the VoiceMon documentation! This project implements production-grade observability for Voice AI agents.

## 📚 Documentation Overview

This repository contains comprehensive documentation to help you understand VoiceMon from multiple angles:

### For Learning & Understanding

1. **[ARCHITECTURE_GUIDE.md](./ARCHITECTURE_GUIDE.md)** (📖 Read First)
   - **Audience**: Developers wanting deep understanding
   - **Content**: Complete technical deep-dive
   - **What you'll learn**:
     - Project overview and problem it solves
     - High-Level Design (system architecture, data models)
     - Low-Level Design (implementation patterns)
     - Line-by-line code explanations
     - Data flows and failure modes
     - Production deployment guide
   - **Time**: 60-90 minutes

2. **[VISUAL_ARCHITECTURE.md](./VISUAL_ARCHITECTURE.md)** (👁️ Visual Learner?)
   - **Audience**: Visual learners, architects
   - **Content**: 15 comprehensive ASCII diagrams
   - **What you'll learn**:
     - System overview and data flows
     - Component internal architectures
     - Patterns like consumer groups, at-least-once delivery
     - Failure recovery scenarios
     - Scaling visualizations
   - **Time**: 30-45 minutes

### For Using & Operating

3. **[QUICK_REFERENCE.md](./QUICK_REFERENCE.md)** (⚡ Quick Start)
   - **Audience**: Developers integrating VoiceMon
   - **Content**: Concise reference guide
   - **What you'll learn**:
     - 3-line integration examples
     - Complete API reference
     - Common queries and patterns
     - Debugging and troubleshooting
     - Production checklist
   - **Time**: 15-20 minutes

4. **[README.md](./README.md)** (🚀 Project Overview)
   - **Audience**: Everyone
   - **Content**: High-level project introduction
   - **What you'll learn**:
     - What VoiceMon does
     - Quick start instructions
     - Features and capabilities
     - Docker setup
   - **Time**: 5-10 minutes

## 🎯 Suggested Reading Paths

### Path 1: "I want to understand everything"
1. Start with [README.md](./README.md) - Get the big picture
2. Read [ARCHITECTURE_GUIDE.md](./ARCHITECTURE_GUIDE.md) - Deep technical dive
3. Review [VISUAL_ARCHITECTURE.md](./VISUAL_ARCHITECTURE.md) - Visualize the concepts
4. Reference [QUICK_REFERENCE.md](./QUICK_REFERENCE.md) - API and usage patterns

**Total time**: 2-3 hours

### Path 2: "I need to integrate VoiceMon ASAP"
1. Skim [README.md](./README.md) - Understand what it does
2. Jump to [QUICK_REFERENCE.md](./QUICK_REFERENCE.md) - Integration examples
3. Reference [ARCHITECTURE_GUIDE.md](./ARCHITECTURE_GUIDE.md) - When you need details

**Total time**: 30 minutes to start, dive deeper as needed

### Path 3: "I'm reviewing the codebase"
1. Read [ARCHITECTURE_GUIDE.md](./ARCHITECTURE_GUIDE.md) - Understand the design
2. Use [VISUAL_ARCHITECTURE.md](./VISUAL_ARCHITECTURE.md) - See the patterns
3. Keep [QUICK_REFERENCE.md](./QUICK_REFERENCE.md) - Handy for API reference

**Total time**: 1-2 hours

### Path 4: "I'm debugging an issue"
1. Check [QUICK_REFERENCE.md](./QUICK_REFERENCE.md) - Debugging tips section
2. Reference [VISUAL_ARCHITECTURE.md](./VISUAL_ARCHITECTURE.md) - Data flow diagrams
3. Deep dive in [ARCHITECTURE_GUIDE.md](./ARCHITECTURE_GUIDE.md) - Component details

**Total time**: Variable, focused reading

## 📖 Key Concepts Covered

Across all documentation, you'll learn:

### Architecture & Design
- ✅ 4-Layer Voice Observability Framework
- ✅ Event-driven architecture with fan-out pattern
- ✅ Redis Streams consumer groups for scalability
- ✅ At-least-once delivery guarantees
- ✅ Rolling window aggregation
- ✅ Z-score anomaly detection
- ✅ Alert cooldown mechanisms
- ✅ OpenTelemetry distributed tracing

### Implementation Details
- ✅ Async-first Python with asyncio
- ✅ Pydantic v2 data models
- ✅ Exporter pattern for pluggable outputs
- ✅ TimescaleDB hypertables for time-series
- ✅ Prometheus histograms and metrics
- ✅ Graceful degradation and fault tolerance

### Operational Practices
- ✅ Horizontal scaling with workers
- ✅ Monitoring the monitoring system
- ✅ Production deployment checklist
- ✅ Common queries and debugging
- ✅ Industry-standard thresholds
- ✅ Cost tracking and optimization

## 🔍 Finding Specific Information

### "How do I...?"

| Question | Document | Section |
|----------|----------|---------|
| Integrate with LiveKit? | QUICK_REFERENCE.md | Integrations → LiveKit |
| Understand the data model? | ARCHITECTURE_GUIDE.md | High-Level Design → Data Models |
| See the event flow? | VISUAL_ARCHITECTURE.md | Diagram 2: Event Flow |
| Configure Redis? | QUICK_REFERENCE.md | Configuration |
| Write PromQL queries? | QUICK_REFERENCE.md | Common Queries → PromQL |
| Debug worker issues? | QUICK_REFERENCE.md | Troubleshooting |
| Scale horizontally? | VISUAL_ARCHITECTURE.md | Diagram 13: Horizontal Scaling |
| Set up alerts? | QUICK_REFERENCE.md | Alert Rules Example |
| Understand collector internals? | ARCHITECTURE_GUIDE.md | Component Explanation → Collector |

### "What is...?"

| Concept | Document | Section |
|---------|----------|---------|
| 4-Layer Framework? | QUICK_REFERENCE.md | Core Concepts Cheat Sheet |
| Redis Streams? | ARCHITECTURE_GUIDE.md | Low-Level Design → Redis Streams |
| Consumer Groups? | VISUAL_ARCHITECTURE.md | Diagram 4: Redis Consumer Groups |
| Z-score anomaly detection? | ARCHITECTURE_GUIDE.md | Low-Level Design → Worker Processor |
| Turn vs Session? | ARCHITECTURE_GUIDE.md | High-Level Design → Data Models |
| At-least-once delivery? | VISUAL_ARCHITECTURE.md | Diagram 12: Failure Recovery |
| Alert cooldown? | VISUAL_ARCHITECTURE.md | Diagram 14: Alert Cooldown |

## 🛠️ Code Examples Locations

All code examples are in **QUICK_REFERENCE.md**:
- Session Management API
- Recording Events (STT, LLM, TTS, UX, Infra, Outcome)
- Exporter Configuration
- Integration Examples (LiveKit, Pipecat, Vapi)
- Common Patterns (cost tracking, A/B testing, etc.)

## 📊 Diagrams Location

All diagrams are in **VISUAL_ARCHITECTURE.md**:
1. System Overview
2. Real-Time Event Flow
3. Query Path
4. VoiceMonCollector Internal Architecture
5. Redis Streams Consumer Group Pattern
6. Worker Processor Internal Flow
7. Data Model Hierarchy (4-Layer)
8. Alert Engine Rule Evaluation Flow
9. TimescaleDB Schema Design
10. Prometheus Metrics Instrumentation
11. OpenTelemetry Span Hierarchy
12. LiveKit Integration Hook Points
13. Failure Recovery (At-Least-Once Delivery)
14. Horizontal Scaling
15. Alert Cooldown Mechanism
16. Anomaly Detection (Z-Score) Visualization

## 🚀 Quick Start

If you're impatient and just want to try it:

```bash
# 1. Clone and start infrastructure
git clone https://github.com/jaiswal-naman/voicemon
cd voicemon
docker compose up -d

# 2. Initialize database
docker compose exec timescale psql -U voicemon -d voicemon -f /schema/schema.sql

# 3. Run demo simulator
python -m tests.demo_simulator --sessions 50 --redis

# 4. Access dashboards
open http://localhost:3000     # Grafana (admin/voicemon)
open http://localhost:8501     # Streamlit Analytics
```

Then read the docs to understand what's happening! 📚

## 💡 Tips for Learning

1. **Don't read linearly** - Jump to what interests you
2. **Run the demo** - See it in action while reading
3. **Check the diagrams** - Visual learners start with VISUAL_ARCHITECTURE.md
4. **Try the code** - Copy examples from QUICK_REFERENCE.md
5. **Ask "why?"** - ARCHITECTURE_GUIDE.md explains design decisions
6. **Refer back** - Keep QUICK_REFERENCE.md open while coding

## 📝 Documentation Statistics

- **Total Lines**: ~2,800 lines
- **Code Examples**: 40+ examples
- **Diagrams**: 15 ASCII diagrams
- **Tables**: 20+ reference tables
- **Coverage**: 100% of core concepts

## 🤝 Contributing

Found a typo or want to improve the docs?
1. Open an issue on GitHub
2. Submit a pull request
3. Suggest missing topics

## 📜 License

All documentation is licensed under Apache-2.0, same as the project code.

---

**Happy Learning! 🎓**

Start with README.md → ARCHITECTURE_GUIDE.md → VISUAL_ARCHITECTURE.md → QUICK_REFERENCE.md
