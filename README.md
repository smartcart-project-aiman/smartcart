# SmartCart

A production-grade, AI-enhanced, multi-vendor B2C e-commerce marketplace.

Built as a structured 9–12 month learning project covering distributed
systems, microservices patterns, event-driven architecture, and AI/ML
engineering.

## What Is This?

SmartCart models the operational domain of a platform like Amazon
Marketplace: independent vendors, a single customer-facing storefront,
and platform-level operations managed by administrators.

**This is a learning project. Not a startup. Not a portfolio checkbox.
A disciplined investment in becoming a senior engineer.**

## Architecture Overview

See [`docs/architecture.md`](docs/architecture.md) for the full
C4 Level 1 and Level 2 diagrams (produced in Phase 1).

## Documentation

| Document | Description |
|----------|-------------|
| [PRD](docs/PRD.md) | Product Requirements Document |
| [ADRs](docs/adr/) | Architecture Decision Records |
| [CONTRIBUTING](CONTRIBUTING.md) | How to contribute |
| [CHANGELOG](CHANGELOG.md) | Version history |

## Project Phases

| Phase | Description | Status |
|-------|-------------|--------|
| 0 | Foundation — repo, standards, tooling | 🟡 In Progress |
| 1 | Requirements & Design — PRD, NFR, ADRs, specs | ⬜ Not Started |
| 2 | Local Infrastructure | ⬜ Not Started |
| 3 | Auth + Product Catalog services | ⬜ Not Started |
| 4 | Remaining core services | ⬜ Not Started |
| 5 | AI/ML services | ⬜ Not Started |
| 6 | Frontend | ⬜ Not Started |
| 7 | Quality Engineering | ⬜ Not Started |
| 8 | CI/CD Hardening | ⬜ Not Started |
| 9 | Cloud Deployment | ⬜ Not Started |
| 10 | Polish & Portfolio | ⬜ Not Started |

## Tech Stack (Summary)

- **Frontend:** React 18 + TypeScript + Vite + Tailwind CSS
- **Backend:** Java (Spring Boot 3, Quarkus), Node.js (NestJS, Fastify), Python (FastAPI, Flask)
- **Databases:** PostgreSQL, MongoDB, Redis, Elasticsearch, ClickHouse, Qdrant
- **Messaging:** Apache Kafka
- **Infrastructure:** Docker, Kubernetes, Helm, ArgoCD, Terraform
- **AI/ML:** LangChain, Qdrant, MLflow
- **Observability:** Prometheus, Grafana, Loki, Jaeger, OpenTelemetry

## Running Locally

> Infrastructure setup is completed in Phase 2. This section will be
> populated then.

## License

MIT
