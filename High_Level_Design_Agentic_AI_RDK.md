# High-Level Design (HLD)
## RDK Agentic AI Platform — Hybrid Cloud/Edge

**Document Version:** 1.0
**Document Type:** High-Level Design
**Governed By:** [Architecture_Vision_Strategy.md](./Architecture_Vision_Strategy.md)
**Implements:** [Final_SRD_Agentic_AI_RDK_Hybrid.md](./Final_SRD_Agentic_AI_RDK_Hybrid.md)
**Audience:** Engineering leads, platform architects, device/cloud implementation teams

---

## 1. Purpose & Scope

This document translates the ratified architecture principles (P1–P8) and decisions (AD-01–AD-09) from the Architecture Vision into a concrete component-level design: what services exist, what they're responsible for, how they talk to each other, and what technology is proposed for each. It does **not** go to module/class/schema-field detail — that is Low-Level Design (LLD), the next phase.

Every component defined here traces back to functional requirement IDs in the SRD (Section 14, Requirement Traceability Matrix) and respects the non-negotiable constraints in Architecture Vision Section 7 (device resource budgets, no on-device LLM/KG/multi-agent runtime, mandatory human gate on irreversible actions).

---

## 2. Design Goals

| Goal | Source |
|---|---|
| Device footprint stays within hard limits regardless of cloud-side feature growth | AD-01, AD-02, SRD §2.3 |
| Cloud tier scales horizontally to 500M+ devices without device-side changes | SRD §9.2 |
| Every component can be independently disabled/rolled back without cross-component failure | P7, AD-06 |
| No single component holds unaudited authority to mutate a subscriber's network | P3, P4, AD-03 |
| Design accommodates operator/HAL variation without forking cloud logic | P8 |

---

## 3. System Context Diagram

```mermaid
flowchart TB
    SUB(["🧑 Subscriber<br/>(Mobile/Web)"])
    CARE(["🎧 Customer Care Agent"])
    NOC(["🛠️ NOC / Engineering"])
    MKT(["📈 Marketing Analyst"])
    SEC(["🛡️ Security Analyst"])
    OPER(["🔧 Platform Administrator"])

    PLATFORM["RDK Agentic AI Platform"]

    GATEWAY["RDK Broadband Gateway Fleet<br/>(ARMv7 class)"]
    OSSBSS["OSS/BSS · CRM ·<br/>Ticketing Systems"]
    OBS["External Observability<br/>(Grafana / Splunk / Elastic)"]
    EXTAI["External / Enterprise<br/>AI Systems"]

    SUB --> PLATFORM
    CARE --> PLATFORM
    NOC --> PLATFORM
    MKT --> PLATFORM
    SEC --> PLATFORM
    OPER --> PLATFORM

    PLATFORM <--> GATEWAY
    PLATFORM <--> OSSBSS
    PLATFORM <--> OBS
    PLATFORM <--> EXTAI

    classDef actor fill:#fff7ed,stroke:#c2410c,color:#7c2d12;
    classDef ext fill:#f8fafc,stroke:#64748b,color:#1e293b;
    classDef core fill:#eef2ff,stroke:#4f46e5,color:#1e1b4b;
    class SUB,CARE,NOC,MKT,SEC,OPER actor;
    class GATEWAY,OSSBSS,OBS,EXTAI ext;
    class PLATFORM core;
```

---

## 4. Component Architecture — Cloud Tier

```mermaid
flowchart TB
    subgraph ING["Ingestion Layer"]
        OTLPGW["OTLP Gateway<br/>HTTP/gRPC receivers"]
        MQ["Streaming Bus<br/>Kafka / Event Hub"]
        OTLPGW --> MQ
    end

    subgraph STORE["Storage Layer"]
        TRACEDB["Trace Store"]
        METRICDB["Metrics Store"]
        LOGDB["Log Store"]
        DL["Data Lake<br/>(historical / ML training)"]
    end

    subgraph KGL["Knowledge & Context Layer"]
        KG["Knowledge Graph<br/>topology & relationships"]
        CACHE["Context Cache"]
    end

    subgraph AI["Reasoning & Agent Layer"]
        RCA["RCA / Reasoning Engine"]
        LLM["LLM Service"]
        AGENTS["Multi-Agent Orchestrator<br/>Network · Device · Care ·<br/>Marketing · Security · Ops"]
    end

    subgraph GOV["Governance Layer"]
        POLICY["Policy Engine"]
        APPROVAL["Approval Workflow Service"]
        AUDIT["Audit & Explainability Store"]
    end

    subgraph API["API & Presentation Layer"]
        GQL["Platform API<br/>REST / GraphQL"]
        DASH["Persona Dashboards"]
    end

    MQ --> TRACEDB & METRICDB & LOGDB --> DL
    MQ --> RCA
    KG --> RCA
    CACHE --> RCA
    RCA --> AGENTS
    AGENTS --> LLM
    AGENTS --> POLICY --> APPROVAL --> AUDIT
    APPROVAL --> GQL --> DASH
    POLICY -- "approved action" --> ACTIONOUT(["→ Device Action Gateway"])
    DL -.trains.-> RCA

    classDef ing fill:#f0f9ff,stroke:#0284c7,color:#0c4a6e;
    classDef store fill:#f8fafc,stroke:#64748b,color:#1e293b;
    classDef kg fill:#fdf4ff,stroke:#a21caf,color:#581c87;
    classDef ai fill:#eef2ff,stroke:#4f46e5,color:#1e1b4b;
    classDef gov fill:#fef3c7,stroke:#d97706,color:#78350f;
    classDef api fill:#ecfdf5,stroke:#059669,color:#064e3b;
    class OTLPGW,MQ ing;
    class TRACEDB,METRICDB,LOGDB,DL store;
    class KG,CACHE kg;
    class RCA,LLM,AGENTS ai;
    class POLICY,APPROVAL,AUDIT gov;
    class GQL,DASH api;
```

### 4.1 Cloud Component Responsibilities

| Component | Responsibility | Key SRD Requirements |
|---|---|---|
| OTLP Gateway | Terminate device OTLP/HTTPS connections; auth; rate-limit per device | FR-TEL-001, NFR-SEC-001/007 |
| Streaming Bus | Decouple ingestion from processing; durable buffering at cloud scale | NFR-SCALE-003, NFR-REL-002 |
| Trace/Metrics/Log Store | Queryable observability backends for engineering + RCA input | FR-TEL-002, NFR-OBS-001 |
| Data Lake | Long-term retention for trend analysis, RCA training, capacity planning | FR-TEL-003, NFR-COMP-001 |
| Knowledge Graph | Subscriber↔Gateway↔Radio↔Device↔Application↔Cloud-Service topology | FR-AI-002 (fills Gap 1 in SRD §14) |
| Context Cache | Low-latency recent-state lookup for reasoning engine | Supports FR-AI-003 latency target |
| RCA / Reasoning Engine | Root-cause analysis, confidence scoring | FR-AI-003/004, NFR-PERF (RCA < 30s) |
| Multi-Agent Orchestrator | Coordinates Network/Device/Care/Marketing/Security/Ops agents | FR-AGENT-001/002/003 |
| LLM Service | Natural-language reasoning, subscriber/care NL interfaces | FR-SS-001, NFR-UX-002 |
| Policy Engine | Risk-based auto-approve vs. escalate decision | FR-AUTO-002, FR-SAFE-001/002 |
| Approval Workflow Service | Human-in-the-loop queueing, multi-stage approval | FR-GOV-004 |
| Audit & Explainability Store | Immutable decision/action/outcome ledger | FR-GOV-001/002/003 |
| Platform API | Unified access for dashboards, mobile app, external AI systems | FR-AGENT-003, NFR-UX |
| Persona Dashboards | Role-specific UI (Subscriber, Care, NOC, Eng, Marketing, Security, Admin) | SRD §7.5 |

---

## 5. Component Architecture — Device Tier

The device tier design is unchanged from the detailed workflow already specified in the SRD; this HLD treats it as one integrated subsystem exposing two external interfaces (telemetry-out, action-in) to the cloud.

```mermaid
flowchart LR
    subgraph GATEWAY["RDK Gateway — Agentic Runtime"]
        direction TB
        SVC["RDK Services<br/>WAN Mgr · OneWiFi · WebPA ·<br/>RFC · RBUS · PAM · SelfHeal · T2"]
        INSTR["OTEL Instrumentation"]
        RULES["Local Rule / Anomaly<br/>Evaluator"]
        COLL["Edge Collector<br/>filter · batch · queue · export"]
        AG["Action Gateway<br/>allowlist · validate · execute"]

        SVC --> INSTR --> RULES
        RULES --> COLL
        RULES -.local trigger.-> AG
        AG --> SVC
    end

    CLOUD_IN(["☁️ Cloud OTLP Gateway"])
    CLOUD_OUT(["☁️ Cloud Policy Engine"])

    COLL == "Telemetry Interface<br/>(OTLP/HTTPS)" ==> CLOUD_IN
    CLOUD_OUT == "Action Interface<br/>(Action API)" ==> AG

    classDef dev fill:#ecfdf5,stroke:#059669,color:#064e3b;
    classDef cloud fill:#eef2ff,stroke:#4f46e5,color:#1e1b4b;
    class SVC,INSTR,RULES,COLL,AG dev;
    class CLOUD_IN,CLOUD_OUT cloud;
```

*Full internal workflow, device-side use-case diagram, and telemetry/action data-flow detail already documented in SRD §6.0–§6.0.2 — not duplicated here.*

---

## 6. Deployment Architecture

```mermaid
flowchart TB
    subgraph REGION1["Cloud Region A — Primary"]
        LB1["Load Balancer / API Gateway"]
        SVC1["Platform Services<br/>Ingestion · Reasoning · Policy"]
        DB1["Primary Data Stores"]
        LB1 --> SVC1 --> DB1
    end

    subgraph REGION2["Cloud Region B — DR / Failover"]
        LB2["Load Balancer / API Gateway"]
        SVC2["Platform Services (standby)"]
        DB2["Replicated Data Stores"]
        LB2 --> SVC2 --> DB2
    end

    subgraph FLEET["Gateway Fleet (Staged Cohorts)"]
        GW1["Cohort 1 — Canary (1%)"]
        GW2["Cohort 2 — Expanding (5–25%)"]
        GW3["Cohort 3 — GA (100%)"]
    end

    GW1 -- "OTLP/HTTPS" --> LB1
    GW2 -- "OTLP/HTTPS" --> LB1
    GW3 -- "OTLP/HTTPS" --> LB1
    DB1 -. "async replication" .-> DB2
    LB1 -. "automatic failover<br/>RTO ≤ 60min" .-> LB2

    classDef primary fill:#eef2ff,stroke:#4f46e5,color:#1e1b4b;
    classDef dr fill:#f8fafc,stroke:#64748b,color:#1e293b;
    classDef fleet fill:#ecfdf5,stroke:#059669,color:#064e3b;
    class LB1,SVC1,DB1 primary;
    class LB2,SVC2,DB2 dr;
    class GW1,GW2,GW3 fleet;
```

**Design notes:**
- Multi-region active/standby satisfies NFR-AVA-001 (99.99%) and NFR-AVA-004 (RPO ≤ 15min, RTO ≤ 60min).
- Gateway cohorts map directly to the SRD §12 phased rollout — cohort membership is a deployment-time property, not a code branch, so the same binary serves canary and GA fleets.
- No device-to-device or device-to-region affinity is required; any gateway can reconnect to any regional endpoint on failover (stateless device-side design, per P1).

---

## 7. Interface Design

### 7.1 Device ↔ Cloud Interfaces

| Interface | Direction | Protocol | Payload | Governing Requirement |
|---|---|---|---|---|
| Telemetry Export | Device → Cloud | OTLP/HTTPS (TLS 1.2+) | Spans, health events (SRD §8 schema) | FR-DEV-002/005, NFR-SEC-001 |
| Action Dispatch | Cloud → Device | Action API (HTTPS, authenticated) | Action recommendation schema (SRD §8) | FR-AUTO-002, FR-SAFE-001 |
| Execution Result | Device → Cloud | Same channel as telemetry export | Action execution schema (SRD §8) | FR-DEV-021 |
| Outcome Verification | Device → Cloud | Same channel | Verification schema (SRD §8) | FR-DEV-021, FR-AUTO-005 |
| Configuration / RFC | Cloud → Device | RFC control channel | Sampling rate, endpoint, kill switch | FR-DEV-023/024 |

All four data-bearing message types share the minimal OTLP schema already defined in SRD §8 — this HLD does not introduce a second schema; it only assigns each schema to a transport interface.

### 7.2 Cloud-Internal Interfaces

| Interface | Between | Style | Notes |
|---|---|---|---|
| Ingestion → Storage | OTLP Gateway / Streaming Bus → Trace/Metric/Log Stores | Async, event-driven | Decouples ingestion spikes from storage write capacity |
| Reasoning → Knowledge Graph | RCA Engine ↔ Knowledge Graph | Synchronous query API | Latency-sensitive (RCA < 30s target) |
| Orchestrator → Agents | Multi-Agent Orchestrator ↔ specialized agents | Internal message bus / RPC | Supports FR-AGENT-002 context/finding exchange |
| Orchestrator → Policy Engine | Agents → Policy Engine | Synchronous request/response | Every recommendation passes through policy before dispatch (P3) |
| Policy → Approval Workflow | Policy Engine → Approval Service | Async (queued) for escalations, sync for auto-approve | FR-GOV-004 multi-stage support |
| Platform API → External Systems | Platform API ↔ OSS/BSS, external AI | REST/GraphQL, authenticated | FR-AGENT-003 interoperability |

---

## 8. Data Architecture

### 8.1 Telemetry Data Model
Governed entirely by the minimal OTLP schema in SRD §8 (device resource attributes, span schema, health-event schema, action/verification schemas). HLD adds no new telemetry fields — schema discipline (P5, one shape per category) is a hard constraint carried forward from the Architecture Vision.

### 8.2 Knowledge Graph Data Model

```mermaid
erDiagram
    SUBSCRIBER ||--o{ GATEWAY : owns
    GATEWAY ||--o{ RADIO : has
    GATEWAY ||--o{ CLIENT_DEVICE : serves
    CLIENT_DEVICE ||--o{ APPLICATION_SESSION : runs
    GATEWAY ||--o{ INCIDENT : reports
    INCIDENT ||--o{ ACTION : triggers
    ACTION ||--o{ OUTCOME : produces
    GATEWAY }o--|| FIRMWARE_VERSION : runs
    GATEWAY }o--|| REGION : located_in
```

This closes **Gap 1 (Knowledge Graph / Service Topology Model)** identified in SRD §14 — without it, the platform can observe ("DNS failures increased") but not reason causally ("DNS failures → buffering → care calls → churn risk").

### 8.3 Storage Tiering & Retention

| Tier | Store | Data | Retention Strategy |
|---|---|---|---|
| Hot | Trace/Metric/Log stores | Recent telemetry for live RCA | Short (days), high query performance |
| Warm | Knowledge Graph + Context Cache | Active topology, recent incidents | Continuously updated, bounded by KG refresh cadence |
| Cold | Data Lake | Historical telemetry, training data | Long-term, configurable per NFR-COMP-001 |
| Immutable | Audit & Explainability Store | Every decision/action/approval record | Retained per compliance policy, append-only |

---

## 9. Technology Stack

Where the Architecture Vision left a decision explicitly open (AD-07/AD-08), this HLD proposes a default and flags it as a **recommendation pending ratification**, not a final choice.

| Layer | Proposed Technology | Status |
|---|---|---|
| Device collector | Custom lightweight C/C++ (not OTEL Collector Go build) | **Adopted** (AD-02) |
| Device↔Cloud transport | OTLP over HTTPS, TLS 1.2+ | **Adopted** |
| Streaming bus | Kafka (or managed equivalent, e.g. Event Hub) | Proposed |
| Trace store | Tempo or Jaeger | **Open — AD-08**, pending backend selection |
| Metrics store | Prometheus/Mimir-compatible | Proposed |
| Log store | Elastic or Loki | Proposed |
| Knowledge Graph | Graph database (e.g., Neo4j) or hybrid graph/relational | **Open — Architecture Vision §8 item 4** |
| LLM / Reasoning | Azure OpenAI, internal LLM, or hybrid | **Open — AD-07** |
| Multi-agent orchestration | Internal orchestrator service (framework TBD at LLD) | Proposed |
| Dashboards/API | REST/GraphQL API + persona-specific web/mobile UI | Proposed |

**Action for architecture review board:** ratify AD-07, AD-08, and the Knowledge Graph technology choice before LLD begins on the Reasoning and Knowledge Graph components — these choices materially affect the interface contracts in §7.2.

---

## 10. Resilience & Scalability Patterns

| Pattern | Applied Where | Purpose |
|---|---|---|
| Backpressure + bounded queue | Device Edge Collector (SRD FR-DEV-008), cloud Streaming Bus | Prevent OOM/overload during traffic bursts or endpoint outage |
| Bounded exponential retry | Device export (FR-DEV-009), cloud→device action dispatch | Avoid tight retry loops (explicit NFR-REL requirement) |
| Circuit breaker | Cloud service-to-service calls (Orchestrator↔Agents↔Policy) | Contain cascading failure across agent services |
| Canary + staged rollout | Fleet deployment (§6), firmware rollout (§12.1 sequence) | Bound blast radius of a bad decision (FR-SAFE-007) |
| Idempotent action execution | Action Gateway (FR-DEV-016) | Safe retry of action dispatch without double-execution |
| Multi-region active/standby | Cloud deployment (§6) | Meets 99.99% availability, RPO/RTO targets |
| Horizontal auto-scaling | Ingestion, Streaming Bus, Reasoning Engine | Meets 500M+ device / 100M+ events-per-hour scale target |

---

## 11. Security Architecture

```mermaid
flowchart LR
    GW["Gateway"] -- "mTLS / OAuth / JWT" --> OTLPGW["OTLP Gateway"]
    OTLPGW -- "least-privilege" --> MQ["Streaming Bus"]
    POLICY["Policy Engine"] -- "signed, replay-protected action" --> AG["Device Action Gateway"]
    AG -- "audit event" --> AUDIT["Audit Store<br/>(append-only)"]
    OTLPGW -- "PII stripped at source<br/>(device-side, SRD FR-SCHEMA-001)" --> STORE["Storage Layer"]

    classDef sec fill:#fef2f2,stroke:#dc2626,color:#7f1d1d;
    class GW,OTLPGW,MQ,POLICY,AG,AUDIT,STORE sec;
```

- **Authentication:** device↔cloud via mTLS/OAuth/JWT (NFR-SEC-007); operator/admin console access via RBAC + MFA (NFR-SEC-003/004).
- **PII handling:** enforced at telemetry *generation*, not filtered downstream — the device never creates the field (NFR-SEC-006, FR-SCHEMA-001). No cloud-side "scrubbing" component is required or trusted as the primary control.
- **Action integrity:** every action carries a unique action ID (idempotency) and is bound to an audit record before dispatch (FR-GOV-003, FR-SAFE-003).
- **Least privilege:** collector and action gateway are separately privileged on-device — a compromised collector cannot execute actions (Architecture Vision §5.5).

---

## 12. Key Interaction Scenarios

### 12.1 Fleet-Wide Firmware Rollout (Canary)

```mermaid
sequenceDiagram
    participant ENG as Device Engineer
    participant ORCH as Agent Orchestrator
    participant POLICY as Policy Engine
    participant CANARY as Canary Cohort (1%)
    participant FLEET as Full Fleet

    ENG->>ORCH: Approve firmware X for rollout
    ORCH->>POLICY: Request canary deployment plan
    POLICY->>CANARY: Deploy firmware X
    CANARY-->>ORCH: Post-update KPI telemetry
    ORCH->>ORCH: Evaluate health vs. baseline
    alt KPIs healthy
        ORCH->>POLICY: Proceed to next cohort
        POLICY->>FLEET: Staged rollout (5% → 10% → 25% → 100%)
    else KPIs degraded
        ORCH->>POLICY: Halt and roll back
        POLICY->>CANARY: Rollback firmware
    end
```

### 12.2 Security Threat Detection & Isolation

```mermaid
sequenceDiagram
    participant DEV as Gateway (Firewall/DNS telemetry)
    participant SECAI as Security Agent
    participant POLICY as Policy Engine
    participant SECAN as Security Analyst
    participant AG as Action Gateway (Device)

    DEV->>SECAI: Anomalous traffic pattern telemetry
    SECAI->>SECAI: Correlate with threat intelligence
    SECAI->>SECAI: Generate threat assessment + confidence score
    SECAI->>POLICY: Recommend device isolation
    POLICY->>SECAN: Escalate (high-risk, human-gated per P3)
    SECAN-->>POLICY: Approve isolation
    POLICY->>AG: Dispatch approved isolation action
    AG->>DEV: Apply quarantine policy
    AG-->>SECAI: Execution result
    SECAI->>DEV: Notify subscriber (transparency, Architecture Vision Risk 6)
```

### 12.3 Cloud Endpoint Outage & Recovery

```mermaid
sequenceDiagram
    participant DEV as Device Collector
    participant Q as Bounded Persistent Queue
    participant CLOUD as Cloud OTLP Gateway

    DEV->>CLOUD: Export telemetry
    CLOUD--xDEV: Endpoint unreachable
    DEV->>Q: Buffer telemetry (bounded, ≤24 MiB per SRD §2.3)
    loop Bounded exponential backoff
        DEV->>CLOUD: Retry export
        CLOUD--xDEV: Still unreachable
    end
    CLOUD->>DEV: Endpoint restored
    DEV->>CLOUD: Drain queue (rate-limited, no CPU/network storm)
    CLOUD-->>DEV: Ack
    Note over DEV,Q: Queue never exceeds configured cap (FR-DEV-008);<br/>oldest-record expiry applies if still full
```

---

## 13. HLD-Level Design Decisions

These are more granular than the Architecture Vision's ADRs and specific to this design pass.

| # | Decision | Rationale |
|---|---|---|
| HLD-01 | Streaming bus sits between OTLP Gateway and storage/reasoning, rather than reasoning consuming directly from the gateway | Decouples ingestion burst handling from downstream processing rate; required at 500M-device scale |
| HLD-02 | Policy Engine is a single choke point for *all* action dispatch, including agent-generated and locally-rule-triggered low-risk actions | Enforces P3/P4 uniformly — no code path bypasses governance, even for "safe" actions |
| HLD-03 | Knowledge Graph is queried synchronously by the Reasoning Engine but updated asynchronously from telemetry | Balances the < 30s RCA latency target against the impracticality of synchronous graph writes at ingestion volume |
| HLD-04 | Device tier exposes exactly two external interfaces (telemetry-out, action-in) — no third-party service talks to the gateway directly | Preserves P8 portability; keeps operator/HAL variation isolated behind the existing device-side abstraction |
| HLD-05 | Audit Store is architecturally separate from the Approval Workflow Service, not a table within it | An approval-service outage or bug must not be able to lose or alter audit history |

---

## 14. Requirement Traceability Matrix

| HLD Component | SRD Requirement Prefixes |
|---|---|
| Device Runtime (Instrumentation, Collector, Rules, Action Gateway) | FR-DEV-xxx, FR-BUD-xxx, NFR-HW-xxx |
| OTLP Gateway / Streaming Bus / Storage Layer | FR-TEL-xxx, NFR-SCALE-xxx, NFR-REL-xxx |
| Knowledge Graph / Context Cache | FR-AI-002, Gap 1 (SRD §14) |
| RCA / Reasoning Engine, LLM Service | FR-AI-xxx |
| Multi-Agent Orchestrator | FR-AGENT-xxx |
| Policy Engine / Approval Workflow / Audit Store | FR-AUTO-xxx, FR-GOV-xxx, FR-SAFE-xxx |
| Platform API / Dashboards | NFR-UX-xxx, SRD §7.5 |
| Deployment (multi-region) | NFR-AVA-xxx |
| Security Architecture | NFR-SEC-xxx, FR-SCHEMA-001 |

---

## 15. Open Items Before LLD

1. Ratify AD-07 (LLM/AI platform), AD-08 (OTEL backend), and Knowledge Graph technology (Architecture Vision §8) — blocks LLD on Reasoning and Knowledge Graph components.
2. Confirm multi-agent orchestration framework/library choice (§9 lists as "TBD at LLD").
3. Define exact Action API contract (request/response schema, error codes, versioning) — currently specified only at the payload-schema level (SRD §8).
4. Define Knowledge Graph refresh cadence and consistency guarantees (HLD-03 assumes async update; exact SLA needs LLD-level definition).
5. Confirm canary cohort sizing/promotion thresholds as concrete, monitored metrics (currently qualitative "KPIs healthy" in §12.1).

---

*This HLD should be reviewed against the Architecture Vision's open decisions (Section 8 there) before component-level LLD work begins. Diagrams use Mermaid and render natively in GitHub, VS Code, and most markdown viewers.*
