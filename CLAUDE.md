# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A static Markdown study guide (~88,000 words) for senior full-stack engineering interview preparation. There is no code, no build system, and no tests — the entire repo is documentation.

## Structure

Content is split across 10 files due to size, plus a README with the full table of contents and the 6-week study plan (the study plan lives in the README, not in a numbered section). Sections are ordered for progressive learning: JS fundamentals first, then TypeScript, then async, then the full frontend block, then backend (including messaging, DSA, and real-time), then DevOps & CI/CD, then observability & security, then cloud & data engineering, then ways of working (Git, engineering excellence, agile, docs), then solution architecture, then generative AI, then engineering leadership & soft skills.

- `01_Foundations_1-3.md` — JavaScript Core, TypeScript Deep Dive, Async/Event-Driven Programming
- `02_Frontend_4-15.md` — Consolidated **Frontend** block: React (fundamentals + 15-19 evolution, re-renders/React Compiler, error boundaries), Hooks, Server Components, Next.js (fundamentals + 12-15 evolution, i18n), Rendering, Routing & Data Fetching (React Router/Remix), State Management, Styling (Modern CSS/Tailwind/CSS-in-JS), Form Management, Build Tools/Monorepos/Performance/Web APIs/Micro-Frontends, Accessibility, Testing
- `03_Backend_16-23.md` — **Backend**: Node.js/Express/NestJS, API Design (REST/GraphQL/gRPC/tRPC/BFF), Auth/AuthZ, Databases (PostgreSQL/MongoDB/Redis/ORMs/Multi-Tenancy), Design Principles, Messaging & Event Streaming (Kafka/RabbitMQ/Kinesis/SQS/SNS/EventBridge), Data Structures & Algorithms, Real-Time Systems
- `04_DevOps_24.md` — DevOps & CI/CD (Docker/K8s/GitHub Actions/GitLab CI/IaC)
- `05_Observability_Security_26-27.md` — Observability (Logs/Metrics/Traces, APM tools), Application Security (OWASP Top 10)
- `06_Cloud_DataEng_28-30.md` — Cloud & Serverless, AWS Services (catalog + deep dives), Data Engineering & ETL (Glue, Airflow/MWAA)
- `07_WaysOfWorking_31-33.md` — **Ways of Working**: Git & Version Control (branching strategies, conventional commits, PR discipline), Engineering Excellence (code review, testing strategy, DORA metrics, incident management), Agile & Delivery (Scrum/Kanban, ceremonies, estimation/story points), Documentation (ADRs, RFCs, runbooks, C4/diagrams)
- `08_SolutionArchitecture_34-36.md` — **Solution Architecture**: Distributed Systems (CAP, consensus, distributed transactions), System Design (interview problems), Solution Architecture (quality attributes/NFRs, trade-offs, RACI, principles)
- `09_GenAI_37.md` — **Generative AI & Prompt Engineering**: LLM fundamentals (tokens/context/temperature/model families), prompting techniques (zero-shot/few-shot/CoT/structured output/prompt injection), RAG (embeddings/vector DBs/chunking), tool use & agentic patterns (ReAct), production considerations (evals, prompt caching, hallucination mitigation, observability)
- `10_Leadership_38.md` — **Engineering Leadership & Soft Skills**: Technical leadership (DRI, leading without authority), mentoring & developing others, giving & receiving feedback (SBI model), stakeholder communication & presenting, meeting facilitation, behavioral interviews & STAR

## Priority Legend

Content is tagged with priority levels: **Critical > Deep > Important > Learn > Know > Refresh**
