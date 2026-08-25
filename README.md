![Rhizome Risk](https://github.com/obrienma/Rhizome-Risk/raw/master/assets/rhizome-risk-logo-purple-nodes.png)

# Rhizome Risk

**The problem:** fraud and compliance detection systems increasingly hand judgment calls to an LLM — "is this transaction risky?" — with no hard floor under that judgment. If the model is wrong, or drifts, or gets a weird input, there's nothing stopping a bad call from becoming a real outcome (a frozen account, a missed fraud pattern, a compliance violation).

**The approach:** Rhizome Risk is a system where the LLM reasons, but never decides alone. Every AI-generated risk judgment has to pass through a deterministic rule or a human-reviewable override before it can affect anything. Fraud/compliance detection in financial transactions and SaaS account activity is the proving ground for this pattern — the architecture itself is domain-agnostic.

> **The LLM reasons — it never owns the decision.**

## How it works, in one example

A SaaS login comes in with a suspicious pattern (say, impossible travel — two logins from different continents minutes apart). Here's the path it takes:

1.  **Xylem-L6** flags the activity using hand-built velocity checks — no ML here, just deterministic rules tracking per-identity state.
2.  **Synapse-L4** picks up the flagged event, validates it against a typed contract, and hands off only what's safe to reason about downstream.
3.  **Sentinel-L7** is where the LLM actually reasons — checking a semantic cache first, then RAG-backed reasoning, then falling back to hard rules if the model's confidence doesn't clear the bar. Its output is a *recommendation*, not an action.
4.  A human or a deterministic rule makes the final call. The LLM's reasoning is visible, logged, and overridable — never the last word.

That four-step path is the whole thesis. Everything below is how it's built and how it's checked.

## The services, in plain terms

Seven services exist because each one owns a different trust boundary — not because more services is inherently better. Splitting them keeps "detect a signal" (Xylem-L6), "normalize and gate what reaches the model" (Synapse-L4), and "let the model reason within a fenced boundary" (Sentinel-L7) as separately testable, separately reasoned-about decisions.

| Service | Stack | What it's for |
| --- | --- | --- |
| **[Sentinel-L7](https://github.com/obrienma/sentinel-l7#readme)** | PHP/Laravel | The compliance engine. Three-tier pipeline (semantic cache → LLM+RAG → rule-based fallback) that evaluates transactions against policy. Exposes an MCP server for direct querying. |
| **[Synapse-L4](https://github.com/obrienma/synapse-l4#readme)** | Python/FastAPI | The gate. Consumes telemetry, validates it, emits typed events — this is the boundary that keeps raw LLM output from ever reaching downstream systems unchecked. |
| **[EventHorizon](https://github.com/obrienma/EventHorizon#readme)** | TypeScript/Fastify | The telemetry pipeline — ingestion, processing, storage, observation. Backed by RabbitMQ and MongoDB, deployed on GKE. |
| **[Xylem-L6](https://github.com/obrienma/Xylem-L6#readme)** | TypeScript/Zod | Ingests SaaS API activity and flags credential stuffing, impossible travel, and scope escalation using hand-built sliding-window checks. |
| **[Arbiter-L8](https://github.com/obrienma/arbiter-l8#readme)** | Python | The evaluation harness — see below. |
| **[Ledger-L5](https://github.com/obrienma/Ledger-L5#readme)** | Python/FastAPI | Usage-based billing off Sentinel-L7 activity. Early-stage. |
| **[Rhizome-Lens](https://github.com/obrienma/Rhizome-Lens)** | OTel/Grafana | Shared observability layer — all other services export traces and metrics here. |

## Architecture

```mermaid
%%{init: {'themeVariables': {'fontSize': '10px'}, 'flowchart': {'nodeSpacing': 15, 'rankSpacing': 25}}}%%
flowchart LR
    EH[EventHorizon]
    XY[Xylem-L6]
    SL[Synapse-L4]
    AR[Arbiter-L8<br/>external eval harness]
    SentinelL7[Sentinel-L7]
    LE[Ledger-L5]
    RL[Rhizome-Lens]

    SL ~~~ AR

    EH -->|Telemetry Events| SL
    XY -->|SaaS Activity| SL
    SL -->|Validated Axioms| SentinelL7
    SentinelL7 -->|Usage Events| LE
    SentinelL7 -->|OTel Traces/Logs| RL
    AR -->|HTTP POST /ingest| SL
    AR -->|MCP: analyze-transaction| SentinelL7
    AR -->|OTel Metrics/Traces| RL

    click EH "https://github.com/obrienma/EventHorizon#readme" "Go to EventHorizon repo"
    click XY "https://github.com/obrienma/Xylem-L6#readme" "Go to Xylem-L6 repo"
    click SL "https://github.com/obrienma/synapse-l4#readme" "Go to Synapse-L4 repo"
    click AR "https://github.com/obrienma/Arbiter-L8#readme" "Go to Arbiter-L8 repo"
    click SentinelL7 "https://github.com/obrienma/sentinel-L7#readme" "Go to Sentinel-L7 repo"
    click LE "https://github.com/obrienma/Ledger-L5#readme" "Go to Ledger-L5 repo"
    click RL "https://github.com/obrienma/Rhizome-Lens#readme" "Go to Rhizome-Lens repo"

    classDef clickable fill:#1d4ed8,stroke:#1e40af,stroke-width:2px,color:#ffffff
    class EH,XY,SL,AR,SentinelL7,LE,RL clickable
```

## Evaluation

**Arbiter-L8** scores every AI-generated verdict against labeled ground truth and live traffic, using a cost-ordered pipeline: cheap heuristics first, escalating to cross-provider disagreement checks, and only calling an LLM judge when those don't resolve it.

**Current numbers, honestly stated:** Sentinel-L7's LLM judge layer scores 92% binary accuracy on a 25-item labeled sample. That sample is small — it's an early checkpoint, not a claim of production-grade reliability, and the plan is to grow it before leaning on the number for anything load-bearing. Full methodology and sample composition are in [Arbiter-L8's README](https://github.com/obrienma/Arbiter-L8#readme).

## Observability

**Rhizome Lens** is the shared Grafana stack — EventHorizon, Synapse-L4, Sentinel-L7, and Arbiter-L8 all export traces and metrics to it via OTLP, so a transaction's path through the whole system is traceable end to end.

<p align="center">
  <img width="75%" height="75%" alt="EventHorizon Grafana dashboard — RED metrics and distributed traces" src="https://github.com/user-attachments/assets/0f2c032c-612b-431d-83b2-f493bf43588c" />
</p>

-   EventHorizon Grafana dashboard — RED metrics and distributed traces
-   Sentinel-L7 operational console — live transaction feed and compliance events UI

## Engineering priorities

Design decisions across the system weigh quality attributes against each other, not toward defaults:

-   **Observability →** Reliability, Resilience
-   **Testability →** Maintainability, Extendability
-   **Security** and **Scalability** are cross-cutting concerns layered across the others.

These trade off against each other — tightening security adds friction to extendability; optimizing scalability early can work against maintainability. Every non-trivial decision is recorded in an Architectural Decision Record (ADR) before it's built; once accepted, ADRs are only reversed by a new one, never edited.

## Further reading

-   [Closing the Loop: What We Actually Shipped from the Roadmap](https://cyberrhizome.ca/blog/10-closing-the-loop-what-we-shipped)
-   [Triple-Defense Idempotency in a Crash-Prone Event Stream](https://cyberrhizome.ca/blog/09-triple-defense-idempotency)
-   [GraphQL Over Four Planes: An ADR That Contradicted Itself](https://cyberrhizome.ca/blog/event-horizon-2026-07-06-graphql-and-the-n-plus-1-we-measured)
-   [all writing →](https://cyberrhizome.ca/blog)