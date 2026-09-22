# Architecture Diagrams

All diagrams are written in **Mermaid**, embedded in Markdown. They render natively on GitHub, so there is nothing to install and nothing to export.

The reason for this choice rather than an image-based tool: a diagram stored as a `.png` drifts from reality within weeks, because updating it means finding the original file, opening a separate application, and re-exporting — so nobody does. Mermaid diagrams are plain text, they are reviewed in the same pull request as the change they describe, and their diffs are readable. A diagram that is wrong is worse than no diagram, and the main defence against wrongness is making updates cheap.

---

## Index

| # | Diagram | Shows | Primary reference |
|---|---|---|---|
| 1 | [High-Level Architecture](01-high-level-architecture.md) | Every component across the client, edge, application, async, data, external and observability layers, with data and control flow | [HLD §4](../HLD.md#4-logical-architecture) |
| 2 | [Authentication Flow](02-authentication-flow.md) | Token lifecycle state machine, registration, login, hot-path verification, refresh rotation with reuse detection, revocation paths | [System Design §1–§3](../SYSTEM_DESIGN.md#1-user-registration) |
| 3 | [Task Management Flow](03-task-management-flow.md) | Task state machine, the authorization decision, list pagination, write-path transactions, concurrent updates, admin cross-user access | [System Design §4–§8](../SYSTEM_DESIGN.md#4-user-accessing-their-tasks) |
| 4 | [Deployment / Infrastructure View](04-deployment-view.md) | Production topology, network segmentation, CI/CD pipeline, environment sizing, provider portability | [HLD §11](../HLD.md#11-infrastructure-view) |

### Diagrams embedded in other documents

| Diagram | Location |
|---|---|
| Entity relationship model | [Data Architecture §2](../DATA_ARCHITECTURE.md#2-entity-relationship-model) |
| Request lifecycle | [HLD §6](../HLD.md#6-request-lifecycle) |
| Defence-in-depth layers | [Security §2](../SECURITY.md#2-defence-in-depth) |
| Scaling evolution path | [Scalability §2](../SCALABILITY.md#2-the-evolution-path) |
| Test suite shape | [Testing Strategy §2](../TESTING_STRATEGY.md#2-shape-of-the-suite) |

---

## Suggested reading order

Diagram 1 for the overall shape, then 2 and 3 for the two flows that define the system's behaviour, then 4 for how it runs. Each diagram page includes a short "reading the diagram" section pointing out the details that are easy to miss and the reasoning behind them.
