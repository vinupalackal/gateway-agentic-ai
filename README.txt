================================================================================
RDK AGENTIC AI PLATFORM — HYBRID CLOUD/EDGE
Project Documentation Index
================================================================================

Last updated: 2026-09-15

--------------------------------------------------------------------------------
1. WHAT THIS PROJECT IS
--------------------------------------------------------------------------------

This folder contains the design documentation for an Agentic AI platform
(inspired by Airties Aura) for ISP/broadband operators running RDK-based
gateways. The platform autonomously observes, reasons about, and remediates
connectivity, device, application, and subscriber-experience issues.

The defining constraint of this project: the target device is a real,
measured, resource-constrained embedded gateway —

    2-core ARMv7, ~750 MB total RAM, ~256 MB available headroom, 32-bit

— not a next-generation chipset. Every design decision in this documentation
set is filtered through that constraint.

Governing principle (see Architecture_Vision_Strategy.md, Section 4):

    "Collect at the edge, think in the cloud, act on the device."

The device is a signal producer and safe action executor only. All AI
reasoning, LLM inference, knowledge graph, and multi-agent orchestration
live in the cloud. No LLM, vector database, knowledge graph, or multi-agent
runtime is deployed on-device under the current hardware roadmap.


--------------------------------------------------------------------------------
2. DOCUMENT HIERARCHY (READ IN THIS ORDER)
--------------------------------------------------------------------------------

The documentation follows a standard SDLC design chain. Each document is
governed by the one above it and implements/elaborates on it. Read top to
bottom the first time through this project.

  [1] Architecture_Vision_Strategy.md
      WHY the architecture is shaped the way it is.
      - Business drivers, vision statement
      - 8 guiding architectural principles (P1-P8)
      - Strategy by domain (device, cloud, telemetry, AI, security, integration)
      - Architecture Decision Log (adopted decisions AD-01..06, open
        decisions AD-07..09 still pending ratification)
      - Non-negotiable constraints that HLD/LLD may not silently override
      - Contains diagrams: current-vs-target transformation, device/cloud
        responsibility split

  [2] Final_SRD_Agentic_AI_RDK_Hybrid.md
      WHAT the system must do — the full Software Requirements Document.
      - Functional requirements, split explicitly into Device-side (FR-DEV-xxx)
        and Cloud-side (FR-AI-xxx, FR-AGENT-xxx, FR-AUTO-xxx, FR-GOV-xxx, etc.)
      - Non-functional requirements incl. the authoritative device resource
        budget (hard RAM/CPU/flash limits, measured against real hardware)
      - RDK component mapping (which existing RDK subsystem backs which
        capability)
      - Minimal OTLP telemetry schema
      - Use cases, user stories, phased rollout plan (Phase 0-5)
      - Test/acceptance criteria and Go/No-Go gates
      - Gap analysis, open assumptions requiring validation
      - Contains diagrams: architecture, device component workflow, device
        use-case diagram, device data-flow, platform use-case diagram,
        end-to-end sequence diagram, rollout-phase diagram
      - This is the largest and most authoritative document — requirement
        IDs here (FR-*, NFR-*) are referenced by every downstream document.

  [3] High_Level_Design_Agentic_AI_RDK.md
      HOW the system is structured at component level.
      - System context diagram
      - Cloud component architecture (Ingestion / Storage / Knowledge Graph /
        Reasoning & Agents / Governance / API layers)
      - Device component architecture (consolidated view; full detail lives
        in the SRD, Section 6)
      - Deployment architecture (multi-region, canary cohorts)
      - Interface design (device<->cloud and cloud-internal interfaces)
      - Data architecture incl. Knowledge Graph entity-relationship diagram
      - Technology stack proposals (flags which choices are still open)
      - Resilience/scalability patterns, security architecture
      - 3 sequence diagrams: firmware rollout (canary), security threat
        isolation, endpoint outage/recovery
      - Requirement traceability matrix back to the SRD

  [4] Low_Level_Design_Agentic_AI_RDK.md
      Implementation-level detail for the device runtime.
      - Full module decomposition: M1 OTEL Instrumentation, M2 Local Rule/
        Health Evaluator, M3 Edge Collector, M4 Action Gateway, M5 RFC/
        Lifecycle Controller
      - State machines for Action Gateway lifecycle, Collector queue,
        RFC enable/disable
      - Algorithms: adaptive sampling, exponential backoff w/ jitter,
        idempotency cache, rule conflict resolution, Policy Engine risk
        scoring
      - Exact message contracts (Telemetry Export, Action Dispatch via
        WebPA/XMidt WRP, Action Result/Verification) with field-level detail
      - Error taxonomy, module-level self-observability metrics
      - Fully implementable for the device side today; cloud-side sections
        are specified as technology-agnostic interface contracts pending
        the open decisions from the Architecture Vision doc

All four documents use Mermaid diagram syntax (```mermaid code blocks),
which renders natively in GitHub, GitLab, VS Code, and most modern markdown
viewers — no separate image files or external tools required.


--------------------------------------------------------------------------------
3. SOURCE MATERIAL (backup/)
--------------------------------------------------------------------------------

The backup/ folder contains the 15 original working documents that the four
documents above were consolidated and synthesized from. Kept for reference
and traceability — if a number or requirement in the main documents looks
wrong, this is where to check the original source.

  airties_Aura_Agentic_AI-Executive_Summary.txt
      Business-level intro to the Aura product this platform is inspired by.

  Requirement_Document.txt
      Early-draft functional requirements (superseded by the consolidated SRD).

  Software_Requirements_Specification.txt
      IEEE-29148-style SRS draft (superseded by the consolidated SRD).

  Functional_And_Non_FunctionalRequirements.txt
      Detailed FR/NFR list (superseded by the consolidated SRD).

  Hardware_Requirements.txt
      Three hardware profile options (cloud-centric / hybrid / full-edge)
      and per-component RAM estimates.

  Hardware_Support_Capability_Accessment.txt
      Feasibility assessment of what the measured gateway can/can't run
      (basis for SRD Section 5).

  Gateway_Capacity_accessment.txt
      Detailed capacity budgets, benchmark test matrix, and acceptance
      criteria (basis for SRD Section 13).

  Measure_Baseline.txt
      The authoritative measured baseline (actual /proc/meminfo output,
      refined resource envelope, full test/benchmark matrix, RDK component
      deployment checklist). This is the single most load-bearing source
      document — its numbers are what the SRD's hard resource limits are
      built on.

  RDK_Component_Mapping.txt
      Maps every Agentic AI capability to the existing RDK subsystem that
      backs it (OneWiFi, WAN Manager, RBUS, WebPA, etc.).

  High_Level_Architecture.txt
      Early hybrid architecture proposal for the ARMv7/750MB gateway class
      (basis for SRD Section 3 and HLD Section 5).

  System_Capabilities_Dependencies.txt
      Six-layer capability/dependency breakdown (telemetry foundation,
      edge collector, context propagation, local AI readiness, storage,
      cloud dependencies).

  Minimal_OTLP_Schema_For_RDK.txt
      The minimal telemetry schema design and phased low-overhead rollout
      plan (basis for SRD Section 8 and Section 12).

  Usecases_And_UserStories.txt
      Actors, detailed use cases (UC-001..007), and Agile user stories
      (US-001..016) (basis for SRD Section 11).

  Gap_Analysis-Agentic_AI_on_RDK.txt
      Architecture gaps, technical/operational risks, and key assumptions
      (basis for SRD Section 14).

  Assumptions_To_Validation.txt
      50 decision-driving validation questions across 12 categories, plus
      the 7-question architecture review gate (basis for SRD Section 15).


--------------------------------------------------------------------------------
4. KEY FACTS AT A GLANCE
--------------------------------------------------------------------------------

  Target hardware .......... 2-core ARMv7, ~750 MB RAM, ~256 MB available,
                              32-bit, DOCSIS gateway
  Device resource envelope . <= 32 MiB steady RSS / 48 MiB peak (target),
                              40/56 MiB hard stop
  Device-side AI ............ deterministic rules + small classical ML only
                              (decision trees, isolation forests) — no LLM
  Cloud-side AI ............. LLM, knowledge graph, multi-agent orchestration,
                              RAG — technology selection still open (see
                              Architecture Vision, Section 8)
  Never-autonomous actions .. factory reset, firmware rollback, mass/fleet
                              config change, firewall-policy replacement —
                              permanently human-gated, not just MVP-gated
  Rollout model .............. Phase 0 (packaging, disabled) through
                              Phase 5 (autonomous safe actions, GA) —
                              see SRD Section 12


--------------------------------------------------------------------------------
5. OPEN DECISIONS BLOCKING FURTHER WORK
--------------------------------------------------------------------------------

These need architecture-review-board ratification before cloud-side LLD/
implementation can proceed (device-side work is NOT blocked by these):

  AD-07  Cloud AI/LLM platform (Azure OpenAI vs. internal LLM vs. hybrid)
  AD-08  OTEL backend selection (Tempo / Jaeger / Grafana / Datadog / Splunk)
  AD-09  MVP governance posture ("AI Suggests, Human Approves" vs.
         "AI Executes, Human Audits")
  --     Knowledge Graph technology (graph DB vs. relational/hybrid)
  --     Multi-agent orchestration framework/library
  --     WebPA/XMidt WRP as the confirmed action-dispatch transport
         (assumed in LLD Section 4.2, not yet ratified with the WebPA
         platform team)

Full detail: Architecture_Vision_Strategy.md Section 8, and the "Open Items"
sections at the end of the HLD and LLD documents.


--------------------------------------------------------------------------------
6. SUGGESTED NEXT STEPS
--------------------------------------------------------------------------------

  - Ratify the open decisions in Section 5 above with the architecture
    review board.
  - Build a requirements-traceability test plan from SRD Section 13
    (Test, Acceptance, and Go/No-Go Criteria).
  - Begin device-side implementation against the LLD — it is not blocked
    by the open cloud-technology decisions.
  - Once AD-07/AD-08/Knowledge-Graph-technology are ratified, complete the
    cloud-side LLD sections that are currently specified only as
    technology-agnostic interface contracts.


================================================================================
End of README
================================================================================
