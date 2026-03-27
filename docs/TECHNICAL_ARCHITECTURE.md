# Agents In Code — Technical Architecture Overview

**Version:** 1.0
**Classification:** Internal Technical Reference
**Last Updated:** July 2025

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [The Compilation Pipeline](#2-the-compilation-pipeline)
3. [Runtime Environment](#3-runtime-environment)
4. [Security Model](#4-security-model)
5. [Observability Stack](#5-observability-stack)
6. [Google Workspace Integration](#6-google-workspace-integration)
7. [Scalability & Performance](#7-scalability--performance)

---

## 1. Architecture Overview

### 1.1 Core Philosophy: Crafted Compilation

Agents In Code operates on a fundamental insight: **the vast majority of AI agent behavior in production is deterministic, repeatable, and expressible as code.** The platform separates the *creative act of designing agent workflows* from the *execution of those workflows*, using AI where it excels (code generation at design time) and eliminating it where it doesn't (repeated inference at runtime).

This approach — **Crafted Compilation** — follows a three-phase model:

1. **Human designs** the agent's workflow, logic, and integration points
2. **AI writes** optimized, production-grade code that implements that design
3. **Code runs** in production — lightweight, fast, and without LLM dependency

The result is a 90%+ reduction in inference costs, sub-second execution times, and enterprise-grade reliability.

### 1.2 High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DESIGN TIME                                  │
│                                                                     │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │              │    │                  │    │                  │  │
│  │  Agent       │───▶│  Compilation     │───▶│  Test & QA       │  │
│  │  Designer    │    │  Pipeline        │    │  Suite           │  │
│  │              │    │                  │    │                  │  │
│  └──────────────┘    └──────────────────┘    └──────────────────┘  │
│                                                       │             │
└───────────────────────────────────────────────────────┼─────────────┘
                                                        │
                                              ┌─────────▼─────────┐
                                              │    Deployment     │
                                              │    Gateway        │
                                              └─────────┬─────────┘
                                                        │
┌───────────────────────────────────────────────────────┼─────────────┐
│                       RUNTIME                         │             │
│                                                       ▼             │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │              │    │                  │    │                  │  │
│  │  Trigger     │───▶│  Agent Runtime   │───▶│  Google          │  │
│  │  Engine      │    │  Engine          │    │  Workspace APIs  │  │
│  │              │    │  (No LLM)        │    │                  │  │
│  └──────────────┘    └──────────────────┘    └──────────────────┘  │
│                              │                                      │
│                     ┌────────▼────────┐                             │
│                     │  Observability  │                             │
│                     │  Stack          │                             │
│                     └─────────────────┘                             │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.3 Design Principles

| Principle | Implementation |
|---|---|
| **AI at design time, code at runtime** | LLM generates code during compilation; production execution is pure code |
| **Zero ambient inference** | No LLM calls in the hot path unless explicitly required by the agent design |
| **Enterprise-first** | SOC 2, audit logging, RBAC, and data residency from day one |
| **Google Workspace native** | First-class integration with Docs, Sheets, Drive, Gmail, and Calendar |
| **Observable by default** | Every agent execution produces structured logs, traces, and metrics |
| **Deterministic reproducibility** | Same input produces same output — every time, verifiably |

---

## 2. The Compilation Pipeline

The compilation pipeline is the heart of the platform. It transforms a human-authored agent design into production-ready, optimized code through a multi-stage process.

### 2.1 Pipeline Stages

```
 ┌───────────┐   ┌───────────┐   ┌───────────┐   ┌───────────┐   ┌───────────┐
 │           │   │           │   │           │   │           │   │           │
 │  DESIGN   │──▶│  ANALYZE  │──▶│  GENERATE │──▶│  VERIFY   │──▶│  DEPLOY   │
 │           │   │           │   │           │   │           │   │           │
 └───────────┘   └───────────┘   └───────────┘   └───────────┘   └───────────┘
      │               │               │               │               │
      ▼               ▼               ▼               ▼               ▼
  Agent Design   Dependency      Generated       Test Results    Versioned
  Specification  Graph &         Code Modules    & Coverage      Artifact
  (ADS)          Cost Model      + Manifest      Report          in Registry
```

### 2.2 Stage 1: Design

The agent designer allows users to express workflows through a structured specification format — the **Agent Design Specification (ADS)**. An ADS captures:

- **Triggers**: What initiates the agent (schedule, webhook, email arrival, document change)
- **Steps**: The ordered sequence of actions the agent performs
- **Conditionals**: Branching logic and decision trees
- **Data mappings**: How data flows between steps, including transformations
- **Integrations**: Which Google Workspace APIs and external services are involved
- **Error handling**: Retry policies, fallback behaviors, and alerting rules
- **LLM escape hatches**: Explicitly marked steps where runtime inference is genuinely required (e.g., unstructured text classification that cannot be rule-based)

```yaml
# Example ADS fragment
agent:
  name: invoice-processor
  version: 2.1.0
  trigger:
    type: gmail.message.received
    filter:
      label: "invoices"
      has_attachment: true

  steps:
    - id: extract_attachment
      action: gmail.get_attachment
      output: raw_pdf

    - id: parse_invoice
      action: document.extract_fields
      input: $raw_pdf
      schema: schemas/invoice_v3.json
      output: invoice_data

    - id: validate
      action: logic.validate
      input: $invoice_data
      rules:
        - field: total
          condition: gte
          value: 0
        - field: vendor_id
          condition: exists_in
          source: sheets.vendor_registry

    - id: record
      action: sheets.append_row
      target: spreadsheets/finance-ledger
      input: $invoice_data
      on_error:
        retry: 3
        fallback: alert.slack
```

### 2.3 Stage 2: Analyze

The analysis stage performs static evaluation of the ADS before any code is generated:

- **Dependency resolution**: Maps all required APIs, schemas, credentials, and data sources
- **Cost modeling**: Estimates per-execution cost and compares against a pure-inference baseline
- **Complexity classification**: Determines which steps can be fully compiled to deterministic code, which require lightweight heuristics, and which genuinely need runtime LLM calls
- **Security surface analysis**: Identifies all permission scopes, data flows, and external touchpoints
- **Conflict detection**: Checks for race conditions, circular dependencies, and incompatible step orderings

The analyzer produces a **Compilation Plan** — a directed acyclic graph (DAG) that the code generator follows.

### 2.4 Stage 3: Generate

This is where AI does its work. The compilation engine invokes LLM-powered code generation to transform each node in the Compilation Plan into production code.

**Key characteristics of the generation process:**

- **Module-per-step generation**: Each step in the ADS becomes an independent, testable module
- **Type-safe interfaces**: Generated code includes full type annotations and interface contracts between modules
- **Idiomatic output**: Code follows language-specific best practices and is formatted to pass linting
- **Deterministic optimization**: The generator identifies patterns that can be compiled to lookup tables, regex matchers, state machines, or simple conditional trees — replacing what would otherwise be LLM inference
- **Workspace SDK integration**: Generated code uses the official Google Workspace client libraries with proper authentication, pagination, and error handling

**Generation is a multi-pass process:**

| Pass | Purpose |
|---|---|
| **Structural pass** | Scaffolds modules, entry points, and the execution harness |
| **Logic pass** | Implements the core logic for each step |
| **Integration pass** | Wires up API clients, authentication, and data serialization |
| **Optimization pass** | Replaces inference-dependent patterns with compiled equivalents |
| **Hardening pass** | Adds error handling, input validation, circuit breakers, and retry logic |

### 2.5 Stage 4: Verify

No generated code reaches production without passing the verification suite:

- **Unit tests**: Auto-generated for each module with both happy-path and edge-case coverage
- **Integration tests**: Executed against sandboxed Google Workspace environments
- **Behavioral tests**: The compiled agent is run against the same inputs as an LLM-inference baseline, and outputs are compared for equivalence
- **Security scan**: Static analysis for credential leakage, injection vulnerabilities, and over-permissioned API calls
- **Performance benchmarking**: Execution time and resource consumption are measured and compared against SLAs

A **Verification Report** is produced and attached to the build artifact. Agents that fail verification are not deployable.

### 2.6 Stage 5: Deploy

Verified agents are packaged as versioned, immutable artifacts and pushed to the **Agent Registry**:

```
agent-registry/
├── invoice-processor/
│   ├── v2.1.0/
│   │   ├── manifest.json          # Metadata, dependencies, permissions
│   │   ├── modules/               # Compiled code modules
│   │   ├── config/                # Environment-specific configuration
│   │   ├── tests/                 # Test suite (shipped with artifact)
│   │   └── verification-report.json
│   ├── v2.0.3/
│   └── v2.0.2/
```

Deployment supports:

- **Blue-green deployments**: New versions run alongside old versions with traffic shifting
- **Canary releases**: Gradual rollout with automatic rollback on error rate thresholds
- **Instant rollback**: Any previous verified version can be promoted in seconds
- **Environment promotion**: Dev → Staging → Production pipeline with gate approvals

---

## 3. Runtime Environment

### 3.1 Design Philosophy

The runtime environment is intentionally **lightweight and LLM-independent**. A deployed agent is compiled code running in a managed execution environment — not a prompt being sent to a model on every invocation.

```
┌─────────────────────────────────────────────────────────┐
│                  Agent Runtime Engine                     │
│                                                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────┐ │
│  │            │  │            │  │                    │ │
│  │  Trigger   │  │  Execution │  │  Integration       │ │
│  │  Manager   │  │  Scheduler │  │  Layer             │ │
│  │            │  │            │  │  (Workspace APIs)  │ │
│  └─────┬──────┘  └──────┬─────┘  └─────────┬──────────┘ │
│        │                │                   │            │
│        └────────┬───────┘                   │            │
│                 ▼                            │            │
│  ┌──────────────────────────┐               │            │
│  │                          │               │            │
│  │  Step Executor           │◀──────────────┘            │
│  │  ┌────┐ ┌────┐ ┌────┐   │                            │
│  │  │ S1 │▶│ S2 │▶│ S3 │   │  S = Compiled Step Module  │
│  │  └────┘ └────┘ └────┘   │                            │
│  │                          │                            │
│  └──────────────────────────┘                            │
│                 │                                        │
│        ┌────────▼────────┐                               │
│        │  State Manager  │                               │
│        │  (per-execution │                               │
│        │   context)      │                               │
│        └─────────────────┘                               │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Core Runtime Components

#### Trigger Manager

Receives and validates inbound events from all supported trigger sources:

| Trigger Type | Source | Mechanism |
|---|---|---|
| **Schedule** | Cron definitions in ADS | Internal scheduler with distributed locking |
| **Webhook** | External HTTP calls | HTTPS endpoint with signature verification |
| **Gmail** | New messages, label changes | Google Pub/Sub push subscription |
| **Drive** | File creation, modification | Drive Activity API + change polling |
| **Calendar** | Event creation, updates | Calendar push notifications |
| **Sheets** | Cell/range changes | Apps Script bridge with webhook relay |
| **Manual** | API call or dashboard | Authenticated REST endpoint |

The Trigger Manager performs deduplication (idempotency keys), rate limiting, and priority classification before forwarding events to the Execution Scheduler.

#### Execution Scheduler

Manages the lifecycle of agent executions:

- **Queue management**: Prioritized execution queues per tenant with configurable concurrency limits
- **Resource allocation**: Assigns execution contexts with appropriate memory and CPU quotas
- **Timeout enforcement**: Hard and soft timeouts per step and per execution
- **Dependency coordination**: For multi-agent workflows, ensures execution ordering and data handoff

#### Step Executor

The core execution loop. For each step in the compiled agent:

1. Load the compiled step module
2. Inject the execution context (input data, credentials, configuration)
3. Execute the step code
4. Validate the output against the declared schema
5. Pass the output to the next step's input binding
6. Record the step result in the execution trace

**Critical distinction**: Step execution is *compiled code execution*, not LLM inference. A step that classifies an email by sender domain runs a lookup table or regex match — not a prompt.

#### State Manager

Maintains per-execution state with the following guarantees:

- **Isolation**: No shared mutable state between concurrent executions
- **Checkpointing**: State is persisted after each step, enabling resume-after-failure
- **TTL management**: Execution state is retained for a configurable period (default: 30 days) for debugging and audit, then garbage collected

### 3.3 LLM Escape Hatch

Not all agent logic can be compiled to deterministic code. Some steps genuinely require language model inference — for example, summarizing an unstructured document or generating a nuanced email response.

For these cases, the ADS supports explicit `llm_required: true` step annotations. These steps:

- Are clearly marked in the Compilation Plan and Verification Report
- Use a managed LLM gateway with model selection, token budgets, and fallback chains
- Are metered separately in the cost model
- Are subject to caching — identical inputs return cached outputs without re-inference
- Are the