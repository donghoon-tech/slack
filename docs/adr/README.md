# Architectural Decision Records (ADR)

This directory contains the decision log for the Slack Clone project.
Decisions are numbered sequentially within their categories to show the evolution of the architecture.

## Core Architecture (01 - 09)
The fundamental design decisions that shape the system's backbone.

| ID | Title | Status | Date |
| :--- | :--- | :--- | :--- |
| [0001](./01-redis-pubsub-broadcasting.md) | **Redis Pub/Sub Broadcasting** | Accepted | 2026-01-10 |
| [0002](./02-full-payload-strategy.md) | **Full Payload Strategy** | Accepted | 2026-01-10 |
| [0003](./03-snowflake-id-ordering.md) | **Snowflake ID Ordering** | Accepted | 2026-01-10 |
| [0004](./04-event-driven-architecture.md) | **Event-Driven Architecture** | Accepted | 2026-01-10 |
| [0005](./05-redis-zset-unread-counts.md) | **Redis ZSET for Unread Counts** | Accepted | 2026-01-10 |
| [0006](./06-async-read-receipts.md) | **Async Read Receipts** | Accepted | 2026-01-10 |
| [0007](./07-redis-zset-presence.md) | **Redis ZSET for Presence** | Accepted | 2026-01-16 |
| [0008](./08-elasticsearch-cdc-search.md) | **ElasticSearch with CDC for Search** | Proposed | 2026-01-31 |
| [0009](./09-gateway-grpc-streaming.md) | **Gateway gRPC Streaming** | Accepted | 2026-01-10 |
| [0010](./10-presence-based-push-suppression.md) | **Presence-Based Push Suppression** | Proposed | 2026-02-09 |

## Implementation Details (50 - 99)
Specific implementation strategies, optimizations, and security patterns.

| ID | Title | Status | Date |
| :--- | :--- | :--- | :--- |
| [0050](./50-explicit-authorization.md) | **Explicit Authorization** | Accepted | 2026-01-05 |
| [0051](./51-redis-pipeline-vs-lua.md) | **Redis Pipeline vs Lua** | Accepted | 2026-01-10 |
| [0052](./52-eventual-consistency-unread.md) | **Eventual Consistency (Unread)** | Accepted | 2026-01-10 |

---

## How to add a new ADR

1.  Copy the [template](0000-template.md).
2.  Choose the next available number in the appropriate category:
    *   **Core**: 07, 08...
    *   **Impl**: 53, 54...
3.  Submit a PR with the new ADR.