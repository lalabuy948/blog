---
title: "Python React to Elixir Phoenix Migration Breakdown"
date: 2026-02-25T14:08:19+01:00
draft: true
comments: true
cover: "img/python-react-to-elixir-phoenix-migration-breakdown/thumbnail.png"
tags: ["elixir", "migration", "code"]
seo: ["elixir phoenix migration", "python to elixir", "react to liveview", "microservices to monolith", "phoenix liveview", "fastapi to phoenix", "elixir migration case study"]
seo_description: "Real-world migration from Python/React microservices to Elixir/Phoenix monolith 63% less code, 9 technologies eliminated, and a 3-person team replaced by 1 Elixir developer."
---

Due to an NDA we can't go into specific business details, but here's the gist: we inherited a SaaS platform in the accounting and document processing space. The system handled document ingestion, automated data extraction, workflow orchestration, and multi-tenant reporting a fairly typical B2B back-office tool. The original stack was maintained by a team of two backend engineers and one frontend developer. What follows is a breakdown of what the migration to Elixir/Phoenix looked like by the numbers.

## Migration Approach

Rather than attempting a gradual, service-by-service port, we took a clean-slate approach. We analyzed the existing MySQL schema to identify the core domain entities (files, accounting entries, companies, users, banks, VAT rates) and their relationships, then scaffolded a fresh Phoenix 1.8 project with `phx.new` and `phx.gen.auth`. DaisyUI and Tailwind v4 gave us a polished UI foundation out of the box. With the data model mapped and auth in place, we had a working MVP with file upload, document processing pipeline, and basic accounting views in under a week.

The following two weeks were spent reaching feature parity: porting the Zeebee BPMN workflows into Oban background jobs, reimplementing the React frontend screens as LiveView pages, and wiring up the same LLM and OCR integrations through Elixir's Req HTTP client. Because Phoenix LiveView eliminated the entire REST API layer and frontend state management, features that previously required coordinating changes across three codebases (backend route, React component, Zeebee worker) could now be built end-to-end in a single Elixir module. The full migration from first commit to functional parity took approximately three weeks.

Let's dive into raw numbers comparison

# Migration Analysis: Numbers

## Architecture Overview

| | **Old Stack** | **New Stack** |
|---|---|---|
| **Architecture** | Microservices (3 separate apps) | Monolith (1 Phoenix app) |
| **Backend** | FastAPI (Python) | Phoenix 1.8 (Elixir) |
| **Frontend** | React 18 + Vite (TypeScript) | Phoenix LiveView (HEEx) |
| **Workers** | Zeebee/Camunda BPMN (Python) | Oban jobs (Elixir) |
| **Database** | MySQL 8.0 | PostgreSQL (SQLite3 for dev) |
| **Cache** | Redis | Not needed (BEAM VM) |
| **Search** | Elasticsearch | Not needed |
| **Job Queue** | Zeebee + Camunda Operate | Oban (in-database) |
| **Languages** | Python + TypeScript + YAML + BPMN | Elixir + minimal JS |

---

## 1. Lines of Code Reduction

| Metric | Old | New | Reduction |
|---|---|---|---|
| **Total application source lines** | **68,850** | **25,185** | **63.4% fewer** |
| **Backend logic (Python vs Elixir)** | 48,615 | 23,716 | **51.2% fewer** |
| **Frontend (TS/JSX vs HEEx + JS)** | 18,797 | 1,330 | **92.9% fewer** |
| **CSS** | 502 | 139 | **72.3% fewer** |
| **Infrastructure config (YAML/Docker/BPMN)** | 5,448 | 439 | **91.9% fewer** |
| **Environment files (.env)** | 262 lines across 5 files | 0 (runtime.exs) | **100% removed** |

**Net reduction: ~43,665 lines of code eliminated** (from 68,850 to 25,185)

---

## 2. Source File Reduction

| Metric | Old | New | Reduction |
|---|---|---|---|
| **Total source files** | **588** | **144** | **75.5% fewer** |
| **Backend files** | 180 Python | 95 Elixir (.ex) | **47.2% fewer** |
| **Frontend files** | 402 TS/JS/TSX | 3 HEEx + 3 JS | **98.5% fewer** |
| **Config/infra files** | 8 YAML + 3 Dockerfiles + 4 docker-compose + 4 conf | 5 config .exs | **73.7% fewer** |

---

## 3. Dependency Reduction

| Metric | Old | New | Reduction |
|---|---|---|---|
| **Backend deps** | 17 Python packages | 28 Mix packages (includes frontend tooling) | Unified |
| **Worker deps** | 147 Python packages (Zeebee) | 0 (included in above) | **100% eliminated** |
| **Frontend deps** | 30 npm + 24 npm dev = **54 packages** | 0 npm packages | **100% eliminated** |
| **Total dependency files** | **218+ packages** across 3 ecosystems | **28 direct / 78 transitive** | **64% fewer** |
| **Package managers** | pip + npm (2) | mix (1) | **50% fewer** |

---

## 4. Infrastructure Simplification

| Metric | Old | New | Reduction |
|---|---|---|---|
| **Docker services required** | **6** (MySQL, Backend, Backend-LLM, Elasticsearch, Zeebe, Operate) | **1** (PostgreSQL) | **83% fewer** |
| **Docker volumes** | 5 (mysql-data, mysql-backup, files, zeebe-data, elastic-data) | 0 | **100% eliminated** |
| **Dockerfiles** | 3 | 0 (optional 1 for prod) | **100% eliminated for dev** |
| **Nginx config files** | 4 .conf files | 1 (simplified) | **75% fewer** |
| **CI/CD pipeline** | GitLab CI (.gitlab-ci.yml, 6.7KB) | Not needed yet | **Simplified** |

---

## 5. Language & Cognitive Complexity

| Metric | Old | New |
|---|---|---|
| **Programming languages** | 4 (Python, TypeScript, YAML, BPMN XML) | 1 (Elixir + minimal JS) |
| **Frameworks to know** | 5 (FastAPI, React, Redux, Zeebee/Camunda, SQLAlchemy) | 1 (Phoenix/LiveView) |
| **Template systems** | JSX/TSX (React) | HEEx (server-rendered) |
| **State management** | Redux Toolkit + React hooks + API calls | LiveView assigns (server state) |
| **API layer** | REST API (53 route files) + gRPC (Zeebe) | None needed (LiveView direct) |
| **Serialization layers** | JSON API requests/responses between services | Direct function calls |
| **Database ORMs** | SQLAlchemy (Python) | Ecto (Elixir) |
| **Migration tools** | Alembic (Python) | Ecto migrations |

---

## 6. Eliminated Entire Technology Layers

The migration **completely removed** these systems:

1. **Camunda Zeebe** - Workflow engine (Java-based, heavy) - replaced by Oban
2. **Elasticsearch** - Was only used as Zeebe exporter - eliminated entirely
3. **Camunda Operate** - Process monitoring UI - replaced by Oban Web dashboard
4. **Redis** - Caching layer - eliminated (BEAM VM handles in-memory state)
5. **React + Redux** - Entire SPA frontend (398 component files) - replaced by LiveView
6. **npm/Node.js** - Frontend build chain - replaced by esbuild + Tailwind (integrated)
7. **gRPC** - Zeebe inter-service communication - eliminated
8. **REST API layer** - 53 API route files - eliminated (LiveView is full-stack)
9. **MySQL** - replaced by PostgreSQL/SQLite3

---

## 7. Developer Experience Benefits

| Area | Old | New |
|---|---|---|
| **Dev startup** | `docker-compose up` (6 services, minutes to boot) | `mix phx.server` (seconds) |
| **Hot reload** | Frontend: Vite HMR / Backend: manual restart | LiveView: automatic on save |
| **Debugging** | 3 separate log streams, gRPC tracing | Single `iex` session, LiveDashboard |
| **Testing** | pytest + Vitest (2 test runners) | ExUnit (1 test runner) |
| **Type safety** | TypeScript (frontend only) | Dialyzer (full stack) |
| **Real-time updates** | Manual WebSocket/polling setup | Built-in LiveView |
| **Pre-commit check** | Multiple linters across ecosystems | `mix precommit` (one command) |

---

## 8. Operational Benefits

| Area | Old | New |
|---|---|---|
| **Memory footprint** | ~2-4GB (6 Docker containers) | ~100-300MB (1 BEAM process) |
| **Deployment units** | 3+ containers minimum | 1 Docker release image |
| **Health monitoring** | Prometheus + Langfuse + Operate UI | LiveDashboard (built-in) |
| **Job monitoring** | Camunda Operate (separate service) | Oban Web (embedded) |
| **Failure recovery** | Complex (cross-service retry, BPMN error handling) | Oban retry with backoff |
| **Concurrency model** | Python async + multiprocessing | BEAM lightweight processes |

---

## Summary Scorecard

| Dimension | Improvement |
|---|---|
| **Total lines of code** | **-63.4%** (68,850 → 25,185) |
| **Total source files** | **-75.5%** (588 → 144) |
| **Frontend code** | **-92.9%** (18,797 → 1,330 lines) |
| **Infrastructure config** | **-91.9%** (5,448 → 439 lines) |
| **Dependencies** | **-64%** (218+ → 78 packages) |
| **Package ecosystems** | **-66%** (3 → 1) |
| **Docker services** | **-83%** (6 → 1) |
| **Programming languages** | **-75%** (4 → 1 primary) |
| **Frameworks to maintain** | **-80%** (5 → 1) |
| **Eliminated technologies** | **9 entire systems removed** |

The migration from a Python/React/Zeebee microservices architecture to a Phoenix monolith delivered a **63% reduction in code**, **76% fewer files**, and eliminated 9 entire technology layers while preserving the same document processing pipeline functionality.

There's another dimension that doesn't show up in line-of-code metrics: compliance. For any SaaS handling accounting data, certifications like ISO 27001, SOC 2, and GDPR audits are inevitable. Every service, dependency, and data flow in the architecture is a line item an auditor needs to review. Going from 6 Docker services to 2, from 218 dependencies across 3 ecosystems to 78 in one, and from 4 languages to 1 drastically shrinks the attack surface and the audit scope. Fewer moving parts means fewer CVEs to track, fewer access control boundaries to document, and a simpler data flow diagram to defend. What used to require mapping cross-service communication over gRPC and REST is now function calls within a single BEAM process. The compliance paperwork practically writes itself.

The original team of two backend engineers and one frontend developer has been replaced by a single Elixir developer maintaining the entire system.

- [1703 Group](https://1703.lu/)
- [Me on X](https://x.com/mrpopov_com)

