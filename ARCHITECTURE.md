# Architecture Reference: NIRIKSHA Environmental Intelligence Network

NIRIKSHA is an offline-capable, distributed environmental intelligence and hazard monitoring platform. This document serves as the high-level technical architecture reference, outlining the end-to-end telemetry pipeline: distributed wireless mesh sensor nodes, dual-protocol ingestion (UART/MQTT), local algorithmic signal conditioning (Kalman filtering, EWMA anomaly detection, hazard fusion), asynchronous database persistence, and real-time dashboard rendering.

For detailed subsystem documentation, refer to the [Technical Documentation Hub](file:///d:/proj/SIH/docs/README.md).

---

## 1. System Topology & Layered Pipeline

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "fontFamily": "Inter, Segoe UI, -apple-system, BlinkMacSystemFont, Roboto, sans-serif",
    "fontSize": "13px",
    "primaryColor": "#ffffff",
    "primaryBorderColor": "#64748b",
    "primaryTextColor": "#0f172a",
    "lineColor": "#475569",
    "clusterBkg": "#f8fafc",
    "clusterBorder": "#94a3b8"
  },
  "flowchart": { "curve": "basis", "htmlLabels": true, "nodeSpacing": 25, "rankSpacing": 40 }
} }%%
flowchart TB
    subgraph EDGE["Field Sensor Nodes (2.4 GHz RF Mesh)"]
        direction LR
        P1["<b>Pole 1: Flood &amp; Submersion</b><br/>ZMPT101B · HC-SR04<br/>SW-520D · MQ Gas Array<br/><i>TX: 2300ms</i>"]
        P2["<b>Pole 2: Grid &amp; Current</b><br/>PZEM-004T (V, I, W, kWh, PF)<br/>SW-520D · MQ Gas Array<br/><i>TX: 2500ms</i>"]
        P3["<b>Pole 3: Gateway Hub</b><br/>painlessMesh Root Sink<br/>Serial UART Bridge (115200)"]
        P1 -.->|"painlessMesh RF"| P3
        P2 -.->|"painlessMesh RF"| P3
    end

    subgraph INGEST["Dual Concurrent Protocols (FastAPI)"]
        direction LR
        SER["<b>PySerial Ingestor</b><br/>Daemon thread · 2s backoff"]
        MQ["<b>Async MQTT Ingestor</b><br/>aiomqtt client (TCP 1883)"]
        NORM["<b>Unified Normalizer</b><br/>Maps payload → Canonical schema<br/>Inactive → NOT_CONNECTED"]
        SER --> NORM
        MQ --> NORM
    end

    P3 ==>|"USB UART Cable (COMx)"| SER

    subgraph PIPELINE["Algorithmic Signal Processing Pipeline"]
        direction LR
        KAL["<b>1D Kalman Filter Bank</b><br/>Tuned Q/R noise rejection"]
        ANOM["<b>Streaming Anomaly Bank</b><br/>EWMA baseline · CUSUM drift"]
        FUS["<b>Hazard Fusion Engine</b><br/>Electrocution &amp; Fire Indices"]
        KAL --> ANOM --> FUS
    end

    NORM -->|"Normalized floats"| KAL

    subgraph FANOUT["Decoupled Fan-Out &amp; Client Distribution"]
        direction LR
        BUF[("<b>Memory Batch Buffer</b><br/>Flush @ 250-300ms or 50 items")]
        WS["<b>WebSocket Hub</b><br/>Thread-safe asyncio broadcaster"]
    end

    FUS -->|"Enriched record"| BUF
    FUS -->|"Real-time stream"| WS

    PG[("<b>PostgreSQL 14+</b><br/>asyncpg pool · executemany")]
    BUF ==>|"Multi-row batch transaction"| PG

    CLIENT["<b>React 19 + Vite Dashboard (Air-Gapped)</b><br/>Sliding window (50-100 pts) · rAF 60 FPS throttler · Web Audio siren"]
    WS ==>|"WebSocket /ws/telemetry"| CLIENT

    classDef default fill:#ffffff,stroke:#64748b,stroke-width:1px,color:#0f172a;
    classDef nodeSafe fill:#f0fdf4,stroke:#16a34a,stroke-width:1.5px,color:#14532d;
    classDef nodeIngest fill:#f8fafc,stroke:#475569,stroke-width:1.5px,color:#0f172a;
    classDef nodeAnalytics fill:#ede9fe,stroke:#7c3aed,stroke-width:1.5px,color:#4c1d95;
    classDef nodeHazard fill:#fef2f2,stroke:#ef4444,stroke-width:1.5px,color:#991b1b;
    classDef nodeClient fill:#eff6ff,stroke:#2563eb,stroke-width:1.5px,color:#1e3a5f;
    classDef nodeStorage fill:#f1f5f9,stroke:#334155,stroke-width:2px,color:#0f172a;

    class P1,P2,P3 nodeSafe;
    class SER,MQ,NORM nodeIngest;
    class KAL,ANOM nodeAnalytics;
    class FUS nodeHazard;
    class BUF,PG nodeStorage;
    class WS,CLIENT nodeClient;
```

---

## 2. Layered Modular Architecture

Every module across the backend and frontend enforces single responsibility and clear separation of concerns:

```
SIH/
├── backend/app/
│   ├── core/
│   │   ├── config.py              # Strongly typed settings from .env (44 lines)
│   │   └── __init__.py
│   ├── db/
│   │   ├── connection.py          # asyncpg pool lifecycle & re-auth (114 lines)
│   │   ├── schema.py              # DDL schema migrations & performance indexes (87 lines)
│   │   ├── buffer.py              # Memory batch buffer & flusher (78 lines)
│   │   ├── telemetry_repo.py      # Telemetry history, stats, and queries (188 lines)
│   │   ├── alerts_repo.py         # Incident persistence, resolution, dedup (169 lines)
│   │   └── __init__.py
│   ├── protocols/
│   │   ├── normalizer.py          # Unified sensor schema transformation (77 lines)
│   │   ├── serial_manager.py      # Crash-resilient PySerial listener (116 lines)
│   │   ├── mqtt_manager.py        # Asynchronous aiomqtt subscriber & publisher (122 lines)
│   │   └── __init__.py
│   ├── analytics/
│   │   ├── kalman.py              # 1D discrete-time linear Kalman filter (59 lines)
│   │   ├── anomaly.py             # Streaming EWMA and CUSUM drift detection (71 lines)
│   │   ├── fusion_engine.py       # Electrocution Risk & Fire Index scoring (154 lines)
│   │   └── __init__.py
│   ├── ai/
│   │   ├── schemas.py             # Pydantic models for chat & Ollama (16 lines)
│   │   ├── diagnostics.py         # Deterministic domain rule heuristics (118 lines)
│   │   ├── ollama.py              # Ollama LLM integration & Markdown audit (85 lines)
│   │   └── __init__.py
│   ├── routers/
│   │   ├── ports.py               # Serial COM listing & baud setup (26 lines)
│   │   ├── mqtt.py                # MQTT client status & configuration (26 lines)
│   │   ├── telemetry.py           # Recent, historical, stats, and CSV export (134 lines)
│   │   ├── alerts.py              # Incident query and resolution endpoints (65 lines)
│   │   ├── websockets.py          # Real-time WebSocket streaming (23 lines)
│   │   ├── ai_assistant.py        # Assistant chat endpoint (85 lines)
│   │   └── __init__.py
│   ├── connection_manager.py      # Thread-safe WebSocket connection registry (39 lines)
│   └── schemas.py                 # Common schemas (16 lines)
├── backend/database.py            # Backward-compatibility facade (46 lines)
├── backend/serial_manager.py      # Backward-compatibility facade (7 lines)
├── backend/mock_simulator.py      # Deprecation stub enforcing zero-mock directive (11 lines)
└── backend/main.py                # Application entrypoint & lifespan (108 lines)
```

---

## 3. Algorithmic Telemetry Processing Pipeline

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "fontFamily": "Inter, Segoe UI, -apple-system, BlinkMacSystemFont, Roboto, sans-serif",
    "fontSize": "13px",
    "primaryColor": "#ffffff",
    "primaryBorderColor": "#64748b",
    "primaryTextColor": "#0f172a",
    "lineColor": "#475569",
    "clusterBkg": "#f8fafc",
    "clusterBorder": "#94a3b8"
  },
  "flowchart": { "curve": "basis", "htmlLabels": true, "nodeSpacing": 25, "rankSpacing": 40 }
} }%%
flowchart TB
    RAW["<b>Raw Transducer Reading (zₖ)</b><br/>Noisy ADC, Ultrasonic echo, Gas conductance"]

    subgraph KALMAN["Stage 1: 1D Discrete-Time Linear Kalman Filter"]
        direction TB
        K_OP["1. Predict: x̂ₖ|ₖ₋₁ = x̂ₖ₋₁|ₖ₋₁, Pₖ|ₖ₋₁ = Pₖ₋₁ + Q<br/>2. Gain: Kₖ = Pₖ|ₖ₋₁ / (Pₖ|ₖ₋₁ + R)<br/>3. Update: x̂ₖ|ₖ = x̂ₖ|ₖ₋₁ + Kₖ(zₖ - x̂ₖ|ₖ₋₁)<br/>4. Covariance: Pₖ|ₖ = (1 - Kₖ)Pₖ|ₖ₋₁"]
    end

    RAW -->|"Noisy ADC sample"| KALMAN

    subgraph ANOMALY["Stage 2: Streaming Anomaly &amp; Drift Engine"]
        direction TB
        A_OP["• EWMA Baseline: μₖ = μₖ₋₁ + α(xₖ - μₖ₋₁)<br/>• Surge Spike Test: |zₖ| = |xₖ - μₖ| / σₖ &gt; 3.0<br/>• Two-Sided CUSUM: Sₖ⁺ = max(0, Sₖ₋₁⁺ + eₖ - k)"]
    end

    KALMAN -->|"Noise-Suppressed x̂"| ANOMALY

    subgraph FUSION["Stage 3: Multi-Sensor Composite Hazard Fusion"]
        direction TB
        F_OP["• Electrocution Risk Index: min(100, V_score × Depth_factor)<br/>• Fire Combustion Index: min(100, Thermal + CO + Combustible)<br/>• Tilt Debounce: Multi-frame persistence check"]
    end

    ANOMALY -->|"Conditioned metric stream"| FUSION

    OUT[("<b>Enriched Telemetry Record &amp; Alert Event</b><br/>Enqueued in buffer &amp; broadcast via WebSocket")]
    FUSION ==> OUT

    classDef default fill:#ffffff,stroke:#64748b,stroke-width:1px,color:#0f172a;
    classDef nodeRaw fill:#f8fafc,stroke:#475569,stroke-width:1.5px,color:#0f172a;
    classDef nodeKalman fill:#ede9fe,stroke:#7c3aed,stroke-width:1.5px,color:#4c1d95;
    classDef nodeAnomaly fill:#eff6ff,stroke:#2563eb,stroke-width:1.5px,color:#1e3a5f;
    classDef nodeHazard fill:#fef2f2,stroke:#ef4444,stroke-width:1.5px,color:#991b1b;
    classDef nodeEnriched fill:#f0fdf4,stroke:#16a34a,stroke-width:2px,color:#14532d;

    class RAW nodeRaw;
    class K_OP nodeKalman;
    class A_OP nodeAnomaly;
    class F_OP nodeHazard;
    class OUT nodeEnriched;
```

### Mathematical Formulations & Tuning:
- **Ultrasonic Depth**: $Q = 0.05, R = 2.0$ (absorbs surface waves while tracking flood level).
- **Gas ADC Array**: $Q = 0.01, R = 0.8$ (smooths sensor heating cycle spikes).
- **Electrical Potential**: $Q = 0.1, R = 0.2$ (fast response to lethal voltage leakage).
- **Electrocution Risk Index ($0 - 100\%$)**: Fuses voltage potential with water depth ($S_{\text{volt}} \times \text{Multiplier}_{\text{depth}}$).
- **Fire & Thermal Combustion Index ($0 - 100\%$)**: Fuses thermal gradient with carbon monoxide (MQ-7) and flammable hydrocarbons (MQ-2).

---

## 4. Ingestion & Storage Decoupling Architecture

To guarantee the backend handles bursts without locking the asyncio loop:
1. **Serial Background Thread**: Dedicated thread reads UART non-blockingly, catching `SerialException` and reconnecting with fixed 2-second backoff.
2. **aiomqtt Ingestion Task**: Concurrent async coroutine subscribing to `niriksha/poles/+/telemetry`.
3. **Decoupled Batch Buffer**: Packets append to an in-memory queue. Flushes occur every 250-300 ms or upon reaching 50 records via `asyncpg.Connection.executemany`.
4. **WebSocket Fan-Out**: Thread-safe async registry broadcasts packets to connected browser tabs without blocking database transactions.

---

## 5. Comprehensive Documentation Matrix

For complete technical specifications, consult the dedicated documentation chapters:

- **[01. System Architecture](file:///d:/proj/SIH/docs/01_SYSTEM_ARCHITECTURE.md)**: End-to-end topology, fault boundaries, and data lifecycles.
- **[02. Hardware & Mesh Network](file:///d:/proj/SIH/docs/02_HARDWARE_AND_MESH.md)**: Microcontroller pinout tables, painlessMesh RF cadence, and sensor calibration math.
- **[03. Ingestion & Protocols](file:///d:/proj/SIH/docs/03_INGESTION_AND_PROTOCOLS.md)**: PySerial thread, aiomqtt client, zero-mock directive, and unified schema normalizer.
- **[04. Analytics & Algorithms](file:///d:/proj/SIH/docs/04_ANALYTICS_AND_ALGORITHMS.md)**: 1D Kalman filter state equations, EWMA, CUSUM drift detection, and hazard fusion scoring.
- **[05. Database & Storage](file:///d:/proj/SIH/docs/05_DATABASE_AND_STORAGE.md)**: PostgreSQL DDL schemas, indexing strategies, and asyncpg batch buffering.
- **[06. Frontend & Dashboard](file:///d:/proj/SIH/docs/06_FRONTEND_AND_DASHBOARD.md)**: React 19 architecture, 60fps RAF throttling, sliding window buffer, and procedural Web Audio sirens.
- **[07. AI Diagnostics Engine](file:///d:/proj/SIH/docs/07_AI_DIAGNOSTICS_ENGINE.md)**: Deterministic heuristic safety rules, Ollama local LLM runtime, and dynamic telemetry injection.
- **[08. Operations & Runbook](file:///d:/proj/SIH/docs/08_OPERATIONS_AND_RUNBOOK.md)**: PowerShell CLI (`manage.ps1`), port mappings, configuration, and troubleshooting runbooks.
- **[09. Modular Architecture](file:///d:/proj/SIH/docs/09_MODULAR_COMPONENTS.md)**: Frontend and backend component hierarchy, sub-file directory breakdown, and single-responsibility guidelines.
