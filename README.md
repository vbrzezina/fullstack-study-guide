# Senior JS Full-Stack Engineer — Complete Knowledge Map & Deep-Dive Guide

---

# Table of Contents

1. [JavaScript Core](./01_Foundations_1-3.md#1-javascript-core) — 40 min
   - [1.1 Closures & Scope](./01_Foundations_1-3.md#11-closures--scope)
   - [1.2 `this` Binding](./01_Foundations_1-3.md#12-this-binding)
   - [1.3 Prototypes & Class Sugar](./01_Foundations_1-3.md#13-prototypes--class-sugar)
   - [1.4 Generators & Iterators](./01_Foundations_1-3.md#14-generators--iterators)
   - [1.5 Proxy & Reflect](./01_Foundations_1-3.md#15-proxy--reflect)
   - [1.6 WeakMap, WeakSet & Symbol](./01_Foundations_1-3.md#16-weakmap-weakset--symbol)
   - [1.7 ES Modules vs CommonJS](./01_Foundations_1-3.md#17-es-modules-vs-commonjs)
   - [1.8 Error Handling Patterns](./01_Foundations_1-3.md#18-error-handling-patterns)
2. [TypeScript Deep Dive](./01_Foundations_1-3.md#2-typescript-deep-dive) — 35 min
   - [2.1 Advanced Types](./01_Foundations_1-3.md#21-advanced-types)
   - [2.2 Generics & Constraints](./01_Foundations_1-3.md#22-generics--constraints)
   - [2.3 Type Guards & Narrowing](./01_Foundations_1-3.md#23-type-guards--narrowing)
   - [2.4 Advanced Patterns](./01_Foundations_1-3.md#24-advanced-patterns)
   - [2.5 TypeScript 5.x Features](./01_Foundations_1-3.md#25-typescript-5x-features)
   - [2.6 Configuration Best Practices](./01_Foundations_1-3.md#26-configuration-best-practices)
3. [Async and Event-Driven Programming](./01_Foundations_1-3.md#3-async-and-event-driven-programming) — 20 min
   - [3.1 The Event Loop](./01_Foundations_1-3.md#31-the-event-loop)
   - [3.2 Async Patterns](./01_Foundations_1-3.md#32-async-patterns)
   - [3.3 Event-Driven Architecture](./01_Foundations_1-3.md#33-event-driven-architecture)
4. [React Fundamentals & Evolution](./02_Frontend_4-15.md#4-react-fundamentals--evolution) — 15 min
5. [Every React Hook Explained](./02_Frontend_4-15.md#5-every-react-hook-explained) — 20 min
   - [5.1 Foundational (React 16.8)](./02_Frontend_4-15.md#51-foundational-react-168)
   - [5.2 Concurrent (React 18)](./02_Frontend_4-15.md#52-concurrent-react-18)
   - [5.3 React 19 Hooks](./02_Frontend_4-15.md#53-react-19-hooks)
6. [Server Components and Suspense](./02_Frontend_4-15.md#6-server-components-and-suspense) — 8 min
7. [Next.js Fundamentals & Evolution](./02_Frontend_4-15.md#7-nextjs-fundamentals--evolution) — 12 min
8. [Rendering Methods Compared](./02_Frontend_4-15.md#8-rendering-methods-compared) — 8 min
9. [Routing, Data Fetching & Server State](./02_Frontend_4-15.md#9-routing-data-fetching--server-state) — 25 min
    - [SWR](./02_Frontend_4-15.md#swr)
    - [TanStack Query](./02_Frontend_4-15.md#tanstack-query)
    - [Data Fetching & Caching Patterns](./02_Frontend_4-15.md#data-fetching--caching-patterns)
10. [Client State Management](./02_Frontend_4-15.md#10-client-state-management) — 20 min
    - [10.1 Built-in Solutions](./02_Frontend_4-15.md#101-built-in-solutions)
    - [10.2 Flux Architecture](./02_Frontend_4-15.md#102-flux-architecture)
    - [10.3 Redux & Redux Toolkit](./02_Frontend_4-15.md#103-redux--redux-toolkit)
    - [10.4 MobX](./02_Frontend_4-15.md#104-mobx)
11. [Styling: Modern CSS, Tailwind, CSS-in-JS](./02_Frontend_4-15.md#11-styling-modern-css-tailwind-css-in-js) — 15 min
    - [11.1 Modern CSS](./02_Frontend_4-15.md#111-modern-css)
    - [11.2 Tailwind CSS v4](./02_Frontend_4-15.md#112-tailwind-css-v4)
    - [11.3 CSS-in-JS, CSS Modules & Zero-Runtime](./02_Frontend_4-15.md#113-css-in-js-css-modules--zero-runtime)
12. [Form Management](./02_Frontend_4-15.md#12-form-management) — 12 min
    - [12.1 Controlled vs Uncontrolled](./02_Frontend_4-15.md#121-controlled-vs-uncontrolled)
    - [12.2 Native Forms + React 19 Actions](./02_Frontend_4-15.md#122-native-forms--react-19-actions)
    - [12.3 React Hook Form](./02_Frontend_4-15.md#123-react-hook-form)
    - [12.4 Formik vs React Hook Form](./02_Frontend_4-15.md#124-formik-vs-react-hook-form)
    - [12.5 Validation: Zod vs Yup](./02_Frontend_4-15.md#125-validation-zod-vs-yup)
13. [Build Tools, Monorepos, Performance & Web APIs](./02_Frontend_4-15.md#13-build-tools-monorepos-performance--web-apis) — 35 min
    - [13.1 Build Tools](./02_Frontend_4-15.md#131-build-tools)
    - [13.2 Monorepos](./02_Frontend_4-15.md#132-monorepos)
    - [13.3 Performance](./02_Frontend_4-15.md#133-performance)
    - [13.4 Web APIs](./02_Frontend_4-15.md#134-web-apis)
    - [13.5 Micro-Frontends](./02_Frontend_4-15.md#135-micro-frontends)
14. [Accessibility (a11y)](./02_Frontend_4-15.md#14-accessibility-a11y) — 15 min
15. [Testing](./02_Frontend_4-15.md#15-testing) — 25 min
    - [15.1 Testing Philosophy](./02_Frontend_4-15.md#151-testing-philosophy)
    - [15.2 Unit Testing](./02_Frontend_4-15.md#152-unit-testing)
    - [15.3 React Testing](./02_Frontend_4-15.md#153-react-testing)
    - [15.4 API Mocking (MSW)](./02_Frontend_4-15.md#154-api-mocking-msw)
    - [15.5 E2E Testing](./02_Frontend_4-15.md#155-e2e-testing)
    - [15.6 Database Testing](./02_Frontend_4-15.md#156-database-testing)
    - [15.7 Testing Best Practices](./02_Frontend_4-15.md#157-testing-best-practices)
    - [15.8 Coverage](./02_Frontend_4-15.md#158-coverage)
    - [15.9 Contract Testing](./02_Frontend_4-15.md#159-contract-testing)
16. [Node.js, Express, NestJS](./03_Backend_16-23.md#16-nodejs-express-nestjs) — 45 min
    - [16.1 Node.js Core](./03_Backend_16-23.md#161-nodejs-core)
    - [16.2 Express](./03_Backend_16-23.md#162-express)
    - [16.3 NestJS](./03_Backend_16-23.md#163-nestjs)
17. [API Design](./03_Backend_16-23.md#17-api-design) — 30 min
    - [17.1 REST](./03_Backend_16-23.md#171-rest)
    - [17.2 GraphQL](./03_Backend_16-23.md#172-graphql)
    - [17.3 gRPC](./03_Backend_16-23.md#173-grpc)
    - [17.4 tRPC](./03_Backend_16-23.md#174-trpc)
    - [17.5 OData](./03_Backend_16-23.md#175-odata)
    - [17.6 Backend for Frontend (BFF)](./03_Backend_16-23.md#176-backend-for-frontend-bff)
    - [17.7 End-to-End Type Safety](./03_Backend_16-23.md#177-end-to-end-type-safety)
18. [Authentication and Authorization](./03_Backend_16-23.md#18-authentication-and-authorization) — 20 min
    - [18.1 JWT (JSON Web Tokens)](./03_Backend_16-23.md#181-jwt-json-web-tokens)
    - [18.2 OAuth 2.0 / OpenID Connect](./03_Backend_16-23.md#182-oauth-20--openid-connect)
    - [18.3 RBAC vs ABAC](./03_Backend_16-23.md#183-rbac-vs-abac)
    - [18.4 Passkeys / WebAuthn](./03_Backend_16-23.md#184-passkeys--webauthn)
    - [18.5 Providers Comparison](./03_Backend_16-23.md#185-providers-comparison)
19. [Databases](./03_Backend_16-23.md#19-databases) — 45 min
    - [19.1 SQL Fundamentals](./03_Backend_16-23.md#191-sql-fundamentals)
    - [19.2 MySQL](./03_Backend_16-23.md#192-mysql)
    - [19.3 PostgreSQL](./03_Backend_16-23.md#193-postgresql)
    - [19.4 ORMs](./03_Backend_16-23.md#194-orms)
    - [19.5 MongoDB](./03_Backend_16-23.md#195-mongodb)
    - [19.6 Redis](./03_Backend_16-23.md#196-redis)
    - [19.7 DynamoDB and Single-Table Design](./03_Backend_16-23.md#197-dynamodb-and-single-table-design)
    - [19.8 Cassandra / Astra DB](./03_Backend_16-23.md#198-cassandra--astra-db)
    - [19.9 Elasticsearch and SOLR](./03_Backend_16-23.md#199-elasticsearch-and-solr)
    - [19.10 Multi-Tenancy Patterns](./03_Backend_16-23.md#1910-multi-tenancy-patterns)
    - [19.11 Caching Strategies](./03_Backend_16-23.md#1911-caching-strategies)
20. [Design Principles](./03_Backend_16-23.md#20-design-principles) — 40 min
    - [20.1 SOLID](./03_Backend_16-23.md#201-solid)
    - [20.2 GRASP Patterns](./03_Backend_16-23.md#202-grasp-patterns)
    - [20.3 GoF Design Patterns](./03_Backend_16-23.md#203-gof-design-patterns)
    - [20.4 Layered Architecture](./03_Backend_16-23.md#204-layered-architecture)
    - [20.5 Enterprise and Resilience Patterns](./03_Backend_16-23.md#205-enterprise-and-resilience-patterns)
21. [Messaging and Event Streaming](./03_Backend_16-23.md#21-messaging-and-event-streaming) — 25 min
    - [21.0 Message Queue vs Event Streaming](./03_Backend_16-23.md#210-message-queue-vs-event-streaming)
    - [21.1 Message Broker Patterns](./03_Backend_16-23.md#211-message-broker-patterns)
    - [21.2 RabbitMQ](./03_Backend_16-23.md#212-rabbitmq)
    - [21.3 Apache Kafka](./03_Backend_16-23.md#213-apache-kafka)
    - [21.4 AWS SQS / SNS](./03_Backend_16-23.md#214-aws-sqs--sns)
    - [21.5 Amazon Kinesis](./03_Backend_16-23.md#215-amazon-kinesis)
    - [21.6 ActiveMQ / Amazon MQ and EventBridge](./03_Backend_16-23.md#216-activemq--amazon-mq-and-eventbridge)
    - [21.7 Choosing a Technology — Side by Side](./03_Backend_16-23.md#217-choosing-a-technology--side-by-side)
    - [21.8 Delivery Guarantees](./03_Backend_16-23.md#218-delivery-guarantees)
22. [Data Structures & Algorithms](./03_Backend_16-23.md#22-data-structures--algorithms) — 15 min
    - [22.1 Essential Patterns](./03_Backend_16-23.md#221-essential-patterns)
23. [Real-Time Systems](./03_Backend_16-23.md#23-real-time-systems) — 8 min
    - [23.1 WebSockets](./03_Backend_16-23.md#231-websockets)
    - [23.2 Server-Sent Events](./03_Backend_16-23.md#232-server-sent-events)
    - [23.3 WebSockets vs SSE](./03_Backend_16-23.md#233-websockets-vs-sse)
24. [DevOps and CI/CD](./04_DevOps_24.md#24-devops-and-cicd) — 30 min
    - [24.1 Docker](./04_DevOps_24.md#241-docker)
    - [24.2 Kubernetes Basics](./04_DevOps_24.md#242-kubernetes-basics)
    - [24.3 CI/CD](./04_DevOps_24.md#243-cicd)
    - [24.4 Infrastructure as Code (IaC)](./04_DevOps_24.md#244-infrastructure-as-code-iac)
25. [Git & Version Control](./07_WaysOfWorking_31-33.md#25-git--version-control) — 20 min
    - [25.1 Branching Strategies](./07_WaysOfWorking_31-33.md#251-branching-strategies)
    - [25.2 Merge Strategies](./07_WaysOfWorking_31-33.md#252-merge-strategies)
    - [25.3 Rebase & History Rewriting](./07_WaysOfWorking_31-33.md#253-rebase--history-rewriting)
    - [25.4 Conventional Commits & semantic-release](./07_WaysOfWorking_31-33.md#254-conventional-commits--semantic-release)
    - [25.5 PR Best Practices](./07_WaysOfWorking_31-33.md#255-pr-best-practices)
    - [25.6 Essential Git Commands](./07_WaysOfWorking_31-33.md#256-essential-git-commands)
26. [Observability](./05_Observability_Security_26-27.md#26-observability) — 15 min
    - [26.1 Three Pillars](./05_Observability_Security_26-27.md#261-three-pillars)
    - [26.2 SLI / SLO / SLA](./05_Observability_Security_26-27.md#262-sli--slo--sla)
    - [26.3 Alerting](./05_Observability_Security_26-27.md#263-alerting)
    - [26.4 APM and Observability Tooling](./05_Observability_Security_26-27.md#264-apm-and-observability-tooling)
27. [Application Security](./05_Observability_Security_26-27.md#27-application-security) — 25 min
    - [27.1 Security Testing Types](./05_Observability_Security_26-27.md#271-security-testing-types)
      - [SAST — SonarQube / SonarCloud](./05_Observability_Security_26-27.md#sonarqube--sonarcloud)
      - [SAST — CodeQL](./05_Observability_Security_26-27.md#codeql)
      - [SAST — Semgrep](./05_Observability_Security_26-27.md#semgrep)
      - [SCA — Snyk](./05_Observability_Security_26-27.md#snyk)
      - [SCA — Trivy](./05_Observability_Security_26-27.md#trivy)
      - [DAST — OWASP ZAP](./05_Observability_Security_26-27.md#dast-dynamic-application-security-testing)
    - [27.2 OWASP Top 10](./05_Observability_Security_26-27.md#272-owasp-top-10)
    - [27.3 Security Headers](./05_Observability_Security_26-27.md#273-security-headers)
28. [Cloud & Serverless](./06_Cloud_DataEng_28-30.md#28-cloud--serverless) — 8 min
    - [28.1 What "Serverless" Actually Means](./06_Cloud_DataEng_28-30.md#281-what-serverless-actually-means)
    - [28.2 Advantages](./06_Cloud_DataEng_28-30.md#282-advantages)
    - [28.3 Drawbacks and When NOT to Use It](./06_Cloud_DataEng_28-30.md#283-drawbacks-and-when-not-to-use-it)
    - [28.4 Serverless vs Containers vs VMs vs PaaS](./06_Cloud_DataEng_28-30.md#284-serverless-vs-containers-vs-vms-vs-paas)
29. [AWS Services](./06_Cloud_DataEng_28-30.md#29-aws-services) — 20 min
    - [29.1 Compute](./06_Cloud_DataEng_28-30.md#291-compute)
    - [29.2 Storage](./06_Cloud_DataEng_28-30.md#292-storage)
    - [29.3 Networking & Edge](./06_Cloud_DataEng_28-30.md#293-networking--edge)
    - [29.4 Data](./06_Cloud_DataEng_28-30.md#294-data)
    - [29.5 Application Integration](./06_Cloud_DataEng_28-30.md#295-application-integration-messaging--orchestration)
    - [29.6 Security & Configuration](./06_Cloud_DataEng_28-30.md#296-security--configuration)
    - [29.7 Search](./06_Cloud_DataEng_28-30.md#297-search)
    - [29.8 Frontend & App Services](./06_Cloud_DataEng_28-30.md#298-frontend--app-services)
    - [29.9 Observability](./06_Cloud_DataEng_28-30.md#299-observability)
    - [29.10 A Serverless Reference Architecture](./06_Cloud_DataEng_28-30.md#2910-a-serverless-reference-architecture)
30. [Data Engineering & ETL](./06_Cloud_DataEng_28-30.md#30-data-engineering--etl) — 12 min
    - [30.1 ETL vs ELT and Core Principles](./06_Cloud_DataEng_28-30.md#301-etl-vs-elt-and-core-principles)
    - [30.2 Batch vs Streaming and Storage Targets](./06_Cloud_DataEng_28-30.md#302-batch-vs-streaming-and-storage-targets)
    - [30.3 AWS Glue](./06_Cloud_DataEng_28-30.md#303-aws-glue)
    - [30.4 Apache Airflow / Amazon MWAA](./06_Cloud_DataEng_28-30.md#304-apache-airflow--amazon-mwaa)
    - [30.5 Orchestration: Glue vs Airflow vs Step Functions](./06_Cloud_DataEng_28-30.md#305-orchestration-glue-vs-airflow-vs-step-functions)
31. [Engineering Excellence](./07_WaysOfWorking_31-33.md#31-engineering-excellence) — 30 min
    - [31.1 Code Craftsmanship & Review](./07_WaysOfWorking_31-33.md#311-code-craftsmanship--review)
    - [31.2 Testing Strategy & Culture](./07_WaysOfWorking_31-33.md#312-testing-strategy--culture)
    - [31.3 Delivery Metrics (DORA)](./07_WaysOfWorking_31-33.md#313-delivery-metrics-dora)
    - [31.4 Reliability & Incident Management](./07_WaysOfWorking_31-33.md#314-reliability--incident-management)
    - [31.5 Behavioral Interviews & STAR](./10_Leadership_38.md#386-behavioral-interviews--star)
    - [31.6 Feature Flags](./07_WaysOfWorking_31-33.md#316-feature-flags)
32. [Agile & Delivery](./07_WaysOfWorking_31-33.md#32-agile--delivery) — 12 min
    - [32.1 Agile Foundations](./07_WaysOfWorking_31-33.md#321-agile-foundations)
    - [32.2 Scrum](./07_WaysOfWorking_31-33.md#322-scrum)
    - [32.3 Kanban](./07_WaysOfWorking_31-33.md#323-kanban)
    - [32.4 Refinement, Grooming & Planning](./07_WaysOfWorking_31-33.md#324-refinement-grooming--planning)
    - [32.5 Estimation & Story Points](./07_WaysOfWorking_31-33.md#325-estimation--story-points)
33. [Documentation](./07_WaysOfWorking_31-33.md#33-documentation) — 12 min
    - [33.1 Documentation Types & Audiences](./07_WaysOfWorking_31-33.md#331-documentation-types--audiences)
    - [33.2 ADRs (Architecture Decision Records)](./07_WaysOfWorking_31-33.md#332-adrs-architecture-decision-records)
    - [33.3 RFCs / Design Docs](./07_WaysOfWorking_31-33.md#333-rfcs--design-docs)
    - [33.4 Runbooks & Operational Docs](./07_WaysOfWorking_31-33.md#334-runbooks--operational-docs)
    - [33.5 Diagrams as Code](./07_WaysOfWorking_31-33.md#335-diagrams-as-code)
34. [Distributed Systems](./08_SolutionArchitecture_34-36.md#34-distributed-systems) — 25 min
    - [34.1 CAP Theorem](./08_SolutionArchitecture_34-36.md#341-cap-theorem)
    - [34.2 Consistency Models](./08_SolutionArchitecture_34-36.md#342-consistency-models)
    - [34.3 Distributed Transactions](./08_SolutionArchitecture_34-36.md#343-distributed-transactions)
    - [34.4 Idempotency](./08_SolutionArchitecture_34-36.md#344-idempotency)
    - [34.5 Distributed Locking](./08_SolutionArchitecture_34-36.md#345-distributed-locking)
    - [34.6 Consistent Hashing](./08_SolutionArchitecture_34-36.md#346-consistent-hashing)
    - [34.7 Cache Strategies](./08_SolutionArchitecture_34-36.md#347-cache-strategies)
35. [System Design](./08_SolutionArchitecture_34-36.md#35-system-design) — 20 min
    - [35.1 Method](./08_SolutionArchitecture_34-36.md#351-method)
    - [35.2 Practice Problems](./08_SolutionArchitecture_34-36.md#352-practice-problems)
    - [35.3 Key Latencies](./08_SolutionArchitecture_34-36.md#353-key-latencies)
36. [Solution Architecture](./08_SolutionArchitecture_34-36.md#36-solution-architecture) — 18 min
    - [36.1 Quality Attributes / NFRs](./08_SolutionArchitecture_34-36.md#361-quality-attributes--nfrs)
    - [36.2 Tradeoffs Between Attributes](./08_SolutionArchitecture_34-36.md#362-tradeoffs-between-attributes)
    - [36.3 Designing Systems & Decision-Making](./08_SolutionArchitecture_34-36.md#363-designing-systems--decision-making)
    - [36.4 RACI & Stakeholder Alignment](./08_SolutionArchitecture_34-36.md#364-raci--stakeholder-alignment)
    - [36.5 Architecture Principles](./08_SolutionArchitecture_34-36.md#365-architecture-principles)
37. [Generative AI & Prompt Engineering](./09_GenAI_37.md#37-generative-ai--prompt-engineering) — 30 min
    - [37.1 LLM Fundamentals](./09_GenAI_37.md#371-llm-fundamentals)
    - [37.2 Prompting Techniques](./09_GenAI_37.md#372-prompting-techniques)
    - [37.3 Retrieval-Augmented Generation (RAG)](./09_GenAI_37.md#373-retrieval-augmented-generation-rag)
    - [37.4 Tool Use & Agentic Patterns](./09_GenAI_37.md#374-tool-use--agentic-patterns)
    - [37.5 Production Considerations](./09_GenAI_37.md#375-production-considerations)
    - [37.6 Claude & Claude Code](./09_GenAI_37.md#376-claude--claude-code)
38. [Engineering Leadership & Soft Skills](./10_Leadership_38.md#38-engineering-leadership--soft-skills) — 20 min
    - [38.1 Technical Leadership](./10_Leadership_38.md#381-technical-leadership)
    - [38.2 Mentoring & Developing Others](./10_Leadership_38.md#382-mentoring--developing-others)
    - [38.3 Giving & Receiving Feedback](./10_Leadership_38.md#383-giving--receiving-feedback)
    - [38.4 Stakeholder Communication & Presenting](./10_Leadership_38.md#384-stakeholder-communication--presenting)
    - [38.5 Meeting Facilitation](./10_Leadership_38.md#385-meeting-facilitation)
    - [38.6 Behavioral Interviews & STAR](./10_Leadership_38.md#386-behavioral-interviews--star)

---

## 6-Week Study Plan

Designed for a senior engineer (9+ years) who needs focused interview preparation, not learning from scratch. The goal is **interview depth** on gaps and modern APIs, not coverage of things you already know well. Section references (§) point to the renumbered guide.

**Time commitment:** 3–4 hours/day weekdays, 2–4 hours weekend days.
**Baseline assumption:** You already build full-stack apps. These weeks sharpen edges, fill gaps, and build interview fluency.

### Week 1: JavaScript Core + TypeScript Mastery

**Goals:** nail the JS fundamentals interviewers probe; reach TypeScript expert level.

- **Day 1–2: JavaScript Core (§1)** — closures (counter, memoize, module pattern), all 4 `this` rules, draw the prototype chain.
- **Day 3–4: TypeScript Advanced (§2)** — mapped/conditional/template-literal types; `infer`; branded types; `satisfies`, `const` type params (TS 5.x).
- **Day 5–6: TypeScript Practice** — 10 [type-challenges](https://github.com/type-challenges/type-challenges); write `Result<T, E>` and `DeepReadonly<T>`; review tsconfig strict flags.
- **Day 7: Review + Build** — tiny strict-typed utility library; quiz the priority tables from §1–§2.

**Deliverables:** ✅ closures/`this`/prototypes fluent without notes ✅ 10 type-challenges ✅ advanced types from memory.

### Week 2: Async + React + Next.js

**Goals:** own the event loop; deep React 18/19 + Next.js 15; routing & data fetching.

- **Day 1: Async & Event Loop (§3)** — draw the loop phases; `process.nextTick` vs microtask vs `setImmediate`; retry w/ backoff, AbortController timeout.
- **Day 2–3: React (§4, §5)** — React fundamentals refresher (reconciliation, render vs commit, rules of hooks); every hook from §5; re-renders & React Compiler, error boundaries (§4); React 19 hooks (`useActionState`, `useOptimistic`, `use()`).
- **Day 4: Server Components + Next.js (§6, §7, §8)** — Server vs Client boundary rules; Next.js fundamentals (file routing, layouts, route handlers, middleware); 4 caching layers; rendering strategies (SSG/ISR/SSR/Streaming/PPR); i18n basics.
- **Day 5: Routing & Data Fetching (§9)** — React Router v6/v7 data APIs, Remix→RR7; request waterfalls, parallel vs sequential, optimistic updates, cache invalidation, prefetch + `<HydrationBoundary>`.
- **Day 6: Build** — Next.js 15 app with Server Components, Server Actions, Suspense streaming, `useOptimistic`; add TanStack Query for a client feature.
- **Day 7: Review** — quiz priority tables from §3–§9.

**Deliverables:** ✅ event-loop diagram from memory ✅ Next.js 15 app w/ Server Actions ✅ every hook explained aloud.

### Week 3: State + Styling + Forms + Testing + a11y

**Goals:** master client/server state; modern styling & forms; deep testing; solid a11y answers.

- **Day 1: State Management (§10)** — Zustand (store, selectors, `devtools`/`persist`); TanStack Query (`useQuery`/`useMutation`, optimistic updates, invalidation); Context vs Zustand vs TanStack Query.
- **Day 2: Styling (§11)** — Tailwind v4 trade-offs; CSS Modules; runtime vs zero-runtime CSS-in-JS (Emotion/styled-components vs vanilla-extract); why runtime CSS-in-JS conflicts with RSC; modern CSS (container queries, `:has()`, `@layer`).
- **Day 3: Forms (§12)** — controlled vs uncontrolled; native FormData + React 19 actions/`useActionState`; React Hook Form + Zod resolver; RHF vs Formik; accessible error messaging.
- **Day 4–5: Testing (§15)** — Vitest (mocks/spies/stubs); React Testing Library (test behavior); MSW (network-level mocks); Playwright E2E (login + form submission).
- **Day 6: Build/Perf/Web APIs + Accessibility (§13, §14)** — Core Web Vitals (LCP/INP/CLS) causes & fixes; code splitting, bundle analysis; ARIA, keyboard nav, focus management, color contrast.
- **Day 7: Review + Build** — add unit + E2E tests to the Week 2 app.

**Deliverables:** ✅ Zustand + TanStack Query integrated ✅ a form built with RHF + Zod ✅ Playwright E2E suite.

### Week 4: Backend — Node.js, APIs, Auth, Databases

**Goals:** NestJS architecture; API design fluency; auth patterns; DB optimization.

- **Day 1: Node.js & NestJS (§16)** — libuv phases; Guards/Interceptors/Pipes/Middleware request lifecycle; NestJS microservices (TCP, Redis transports).
- **Day 2: API Design (§17)** — REST (HATEOAS, versioning, idempotency, pagination); GraphQL N+1 → DataLoader; gRPC vs REST vs GraphQL; BFF vs API Gateway distinction; write an OpenAPI spec.
- **Day 3: Auth & Authorization (§18)** — JWT structure/expiry/refresh rotation; OAuth 2.0 (Auth Code + PKCE vs Client Credentials); RBAC vs ABAC guard.
- **Day 4–5: Databases (§19)** — PostgreSQL `EXPLAIN ANALYZE`, index types, window functions/CTEs/lateral joins; Redis use cases (cache, pub/sub, rate-limit, lock); MongoDB aggregation.
- **Day 6: Practice** — window-function query + EXPLAIN; Redis rate-limiter; paginated/filterable REST resource.
- **Day 7: Review.**

**Deliverables:** ✅ JWT auth w/ refresh rotation ✅ 3 optimized PostgreSQL queries ✅ Redis caching layer.

### Week 5: Architecture + DevOps + Cloud + Security

**Goals:** design patterns fluency; Docker/K8s ops; serverless & AWS; security & observability.

- **Day 1: Design Principles (§20)** — SOLID (one example each); Factory/Strategy/Observer/Decorator/Repository; Circuit Breaker, Saga, CQRS.
- **Day 2: Distributed Systems (§34)** — CAP examples; consistent hashing, saga, distributed transactions; idempotency keys, exactly-once.
- **Day 3: Messaging (§21)** — queue vs streaming (consume-and-delete vs durable log); Kafka partitions/consumer groups/offsets/compaction; Rabbit vs Kafka vs Kinesis vs SQS/SNS vs EventBridge; DLQs, redrive, outbox.
- **Day 4: DevOps + Cloud/Serverless (§24, §28, §29)** — Docker multi-stage build; K8s Pod/Deployment/Service/ConfigMap/HPA; CI/CD (GitHub Actions *and* `.gitlab-ci.yml`); IaC CDK vs Terraform vs CloudFormation; serverless scale-to-zero/cold starts (§28); AWS core services — Lambda, API Gateway, S3, DynamoDB, SQS/SNS, EventBridge, Step Functions (§29).
- **Day 5: Observability + Security (§26, §27)** — structured logging + correlation IDs; RED metrics; OpenTelemetry tracing; OWASP Top 10 (SQLi, XSS, CSRF, SSRF, broken auth) + fixes; JWT pitfalls (`alg:none`).
- **Day 6: Solution Architecture (§36) + Generative AI (§37) + System Design Practice (§35)** — quality attributes/NFRs (the -ilities); the key trade-offs (availability vs consistency, scalability vs simplicity); ranking attributes by business priority; LLM fundamentals (tokens, context, temperature), prompting techniques (CoT, few-shot, structured output), RAG pipeline (embeddings, vector DB, chunking), tool use & agentic patterns (ReAct), production concerns (evals, prompt caching, observability); then design 1 system (URL shortener or rate limiter): requirements → capacity → components → data model → bottlenecks.
- **Day 7: System Design Practice (§35)** — design a 2nd system (notification service); practice stating NFR trade-offs aloud and recording the decision as an ADR (§33.2).

**Deliverables:** ✅ SOLID w/ 5 examples ✅ 2 system designs end-to-end (with explicit NFR trade-offs) ✅ Docker multi-stage + K8s Deployment from memory.

### Week 6: DSA + System Design + Ways of Working + Mock Interviews

**Goals:** 20–30 LeetCode Mediums (patterns); system design fluency; ways-of-working fluency (eng practices, agile, docs); behavioral polish.

- **Day 1–2: DSA Core Patterns (§22)** — two pointers (3), sliding window (3), BFS/DFS trees (3), binary search (2), hash maps (2). Pattern recognition, not memorization.
- **Day 3: DSA Advanced** — 1D DP (coin change, climbing stairs, house robber); linked list (reverse, cycle, merge); heap / top-K.
- **Day 4: System Design Deep Dive** — review §35 framework; design 1 read-heavy + 1 write-heavy system; capacity estimation, trade-offs, failure modes.
- **Day 5: Ways of Working + Leadership + Behavioral (§25, §31, §32, §33, §38)** — Git: branching strategies (TBD vs GitFlow vs GitHub Flow), conventional commits + semantic-release, PR best practices (§25); Engineering Excellence (code review, test pyramid, DORA metrics, blameless postmortems/incident lifecycle §31); Agile (Scrum vs Kanban, refinement vs planning, story points = relative effort §32); Documentation (ADRs — context + rejected alternatives, RFC vs ADR §33); Leadership & Soft Skills (leading without authority, SBI feedback model, stakeholder communication, meeting facilitation §38); Behavioral Interviews & STAR (§38.6): 8 must-have story themes, senior leveling signals, questions to ask. Write 5 STAR stories; prepare 5 questions for interviewers.
- **Day 6–7: Mock Interviews** — timed coding (2 Mediums) + system design aloud (state NFR trade-offs, §36); behavioral + ways-of-working questions ("how do you review a PR / handle an incident / estimate?") + weak-area review.

**Deliverables:** ✅ 25+ Mediums ✅ 4 system designs ✅ 5 STAR stories ✅ can answer eng-practice/agile/architecture-trade-off questions ✅ ready to interview.

### Compressed 4-Week Version

If you have only 4 weeks, cut these lower-priority areas:
- Styling / Accessibility (§11, §13–§14) — review tables only
- Git (§25) — you know this already
- Real-Time Systems (§23) — skim only
- Agile & Documentation (§32–§33) — review tables only (keep Engineering Excellence §31 and Solution Architecture §36 for senior signal)
- XState, contract testing, visual regression (low interview frequency)

**4-week priority order:** (1) JS Core + TypeScript; (2) Async + React + Next.js + routing/data; (3) Backend + Databases + Auth (combine); (4) Architecture (incl. §36 NFR trade-offs) + Engineering Excellence (§31) + DSA + Mock Interviews.

### Priority Coverage Matrix

| Area | §§ | Interview Frequency |
|---|---|---|
| JavaScript Core + TypeScript | 1–2 | **Very High** |
| Async + React + Next.js + Routing | 3–9 | **Very High** |
| State + Styling + Forms | 10–12 | **High** |
| Build/Perf + Accessibility + Testing | 13–15 | **Medium–High** |
| Backend + APIs + Auth + DBs + Design Principles | 16–20 | **Very High** |
| Messaging + DSA + Real-Time | 21–23 | **High / Medium** |
| DevOps + CI/CD | 24 | **High** |
| Observability + Security | 26–27 | **High** |
| Cloud + Serverless + AWS | 28–29 | **High** |
| Data Engineering + ETL | 30 | **Medium** |
| Engineering Excellence | 31 | **High** |
| Agile & Docs | 32–33 | **Medium** |
| Distributed Systems + System Design + Solution Architecture | 34–36 | **High** |
| Generative AI & Prompt Engineering | 37 | **High** |
| Engineering Leadership & Soft Skills | 38 | **High** |

---

## Coverage Summary

| Category          | Coverage | Priority Topics                           |
| ----------------- | -------- | ----------------------------------------- |
| **JS + TypeScript** | Complete | Closures, prototypes, advanced types    |
| **Frontend**      | Complete | React 19, Next.js 15, Routing/Data, Styling, Forms, a11y, Testing, Micro-Frontends |
| **Backend**       | Complete | Node.js, NestJS, APIs, BFF, Databases, Multi-Tenancy, Messaging, DSA, Real-Time |
| **DevOps**        | Complete | Docker, K8s, CI/CD (GitHub/GitLab), IaC |
| **Observability & Security** | Complete | Logs/Metrics/Traces, APM, OWASP Top 10 |
| **Cloud + Serverless** | Complete | Serverless model, AWS service catalog + deep dives |
| **Data Engineering** | Complete | ETL/ELT, Glue, Airflow, lake/warehouse/lakehouse |
| **Git & Version Control** | Complete | TBD vs GitFlow vs GitHub Flow, conventional commits, semantic-release, PR discipline |
| **Engineering Excellence** | Complete | Code review, testing strategy, DORA metrics, incident management, behavioral interviews & STAR |
| **Agile & Delivery** | Complete | Scrum/Kanban, ceremonies, estimation & story points |
| **Documentation** | Complete | ADRs, RFCs/design docs, runbooks, C4/diagrams |
| **Solution Architecture** | Complete | Distributed Systems, System Design, Quality attributes/NFRs, trade-offs, RACI |
| **Generative AI** | Complete | LLM fundamentals, prompting (CoT/RAG/tool use), production concerns |
| **Leadership & Soft Skills** | Complete | Technical leadership, mentoring, SBI feedback, stakeholder comms, STAR |

---

_Total Guide Size: ~88,000 words across 38 sections (10 parts)_
_Last Updated: June 2026_
