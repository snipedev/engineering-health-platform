# Architecture

## Overview

The **Engineering health Platform** is a backend‑only system designed to transform raw GitHub engineering activity into actionable delivery and productivity metrics for engineering leadership.
The system follows a **clear separation of concerns:**

- **A batch ingestion & metrics engine (Go)** responsible for data collection and computation
- **A serving API layer (.NET)** responsible for exposing metrics to downstream consumers

This separation mirrors real‑world platform architectures where ingestion and serving workloads have different scaling, reliability, and evolution characteristics.

## High-level Architecture

+-------------+        +-------------------------+
| GitHub API  | -----> | Go Ingestion & Metrics  |
+-------------+        | Engine (Batch Job)      |
                       +-----------+-------------+
                                   |
                                   v
                           +---------------+
                           | PostgreSQL    |
                           | (Metrics DB)  |
                           +-------+-------+
                                   |
                                   v
                         +-------------------+
                         | .NET REST API     |
                         | (Read-Only)       |
                         +-------------------+
``

## Core Components

## 1. GO Ingestion & Metrics Engine

### purpose

- Fetch raw github data(PRs, reviews, merges)
- Normalize events into a  consistent internal model
- Compute delivery and flow metrics
- Persist aggregated results
  
### Characteristics

- Batch‑oriented, not continuously running
- Executed via CLI or scheduler (e.g., cron, CI job)
- Idempotent by design
- No externally exposed HTTP APIs

### Why Go

- Efficient concurrency for I/O‑bound GitHub API calls
- Simple, self‑contained deployments
- Clear distinction between compute workloads and serving workloads

## 2. PostgreSQL (Metrics Store)

### Purpose

- Persist aggregated and query‑optimized metrics
- Act as the single source of truth between ingestion and API layers

### Why PostgreSQL

- Strong consistency guarantees
- Rich querying capabilities for time‑based metrics
- Familiar operational model for engineering teams

### Data Characteristics

- Append‑oriented or periodic recomputation
- No real‑time mutation requirements
- Optimized for reads by the API layer

## 3. .NET REST API

## Usage

- Expose computed metrics to consumers
- Translate raw metric data into business‑readable responses
- Act as the stable contract for future dashboards or integrations

### Characteristics

- Read‑only API (no mutation endpoints in v1)
- Explicit versioning and contracts
- No frontend responsibility

### Why .NET

- Strong typing and contract clarity
- Enterprise familiarity
- Easy future extension for authentication, RBAC, and governance

### Data Flow

- The ingestion engine is triggered manually or via a scheduler
- GitHub data is fetched incrementally for configured repositories
- Events are normalized and transformed into domain models
- Delivery metrics are computed in batch
- Results are written to PostgreSQL
- The .NET API serves metrics via REST endpoints

This flow deliberately avoids tight coupling or synchronous dependencies between ingestion and serving.

## Key Design Principles

### API‑First, UI‑Agnostic

The system exposes metrics via APIs only. Dashboards or visualizations are considered downstream concerns and are not part of the core platform.

### Clear Responsibility Boundaries

- Go service: ingestion, computation, correctness
- .NET service: contracts, abstraction, consumption

This reduces cognitive load and enables independent evolution.

### Decision‑Oriented Metrics

Metrics are designed to answer **engineering management questions**, not to track individual performance.

## Non‑Goals

- Real‑time or near‑real‑time metrics
- Individual performance evaluation
- Complex data science or ML‑based insights
- Full GitHub application replacement
- UI‑heavy dashboards

Explicitly stating non‑goals helps prevent scope creep and keeps the system focused.

## Scalability Considerations (Future)

- Horizontal scaling of ingestion via repository sharding
- Incremental ingestion using cursors or checkpoints
- Read replicas for API traffic
- Materialized views for frequently accessed metrics

None of these are required for v1 but are compatible with the current design.

## Failure Modes & Resilience

- Ingestion failures do not impact API availability
- Partial ingestion runs can be safely retried
- External API rate limits are handled in the ingestion layer
- Clear logs and run‑level diagnostics are preferred over complex recovery logic

## Summary

This architecture intentionally prioritizes:

- clarity over cleverness
- separation over convenience
- decision support over raw data exposure

The result is a small but realistic platform that reflects how engineering organizations build and operate internal metrics systems.
