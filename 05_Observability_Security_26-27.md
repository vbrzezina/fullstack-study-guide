# Observability & Security (26-27)
### Observability, APM, Application Security

---

# 26. Observability

Monitoring tells you *whether* a system is healthy; **observability** is the property of being able to ask *why* it's behaving the way it is — including questions you didn't anticipate when you instrumented it. The distinction matters at senior level: a dashboard of pre-defined metrics catches known failure modes, but novel production incidents ("checkout is slow, but only for users in EU on mobile") require the ability to slice and correlate rich telemetry after the fact. The foundation is the **three pillars** — logs, metrics, and traces — increasingly unified under **OpenTelemetry (OTel)**, a vendor-neutral standard for generating and exporting telemetry so you're not locked into one backend. Above the raw data sit the practices that make it actionable: SLI/SLO/error budgets (how you define and defend reliability) and alerting (how you get woken up for the right reasons).

## 26.1 Three Pillars

The three pillars answer different questions and are strongest in combination. **Logs** are discrete, timestamped records of events — the detailed "what happened" you read when debugging a specific request. **Metrics** are numeric aggregates over time (request rate, error count, latency percentiles) — cheap to store and ideal for dashboards and alerts, but they tell you *that* something is wrong, not *which* request. **Traces** follow a single request across service boundaries, breaking it into timed spans — they tell you *where* the time went in a distributed call chain. The senior move is correlating them: a metric alert fires, you pivot to the traces for the slow requests, then drill into the logs for one trace via a shared correlation/trace ID.

### Logs

Structured logs (JSON) are machine-readable and filterable — the first tool you reach for when debugging a specific request, error, or user action. Include a **correlation/trace ID** on every log line so you can find all events for a single request across services.

**What happened**

```typescript
import pino from "pino";

const logger = pino({
  level: process.env.LOG_LEVEL || "info",
});

// Structured logging
logger.info({ userId: 123, action: "login" }, "User logged in");
logger.error({ err, userId: 123 }, "Failed to process payment");
```

**Best practices:**

- Use structured logging (JSON)
- Include correlation ID
- Log levels: trace, debug, info, warn, error, fatal
- Don't log sensitive data (PII, passwords)

### Metrics

**How much / how fast**

#### RED Method (Services)

- **Rate**: Requests per second
- **Errors**: Error rate
- **Duration**: Response time (p50, p95, p99)

```typescript
import client from "prom-client";

// Counter
const httpRequestsTotal = new client.Counter({
  name: "http_requests_total",
  help: "Total HTTP requests",
  labelNames: ["method", "route", "status_code"],
});

app.use((req, res, next) => {
  res.on("finish", () => {
    httpRequestsTotal.inc({
      method: req.method,
      route: req.route?.path || req.path,
      status_code: res.statusCode,
    });
  });
  next();
});

// Histogram (duration)
const httpRequestDuration = new client.Histogram({
  name: "http_request_duration_seconds",
  help: "HTTP request duration",
  labelNames: ["method", "route", "status_code"],
  buckets: [0.1, 0.5, 1, 2, 5],
});

// Expose metrics endpoint
app.get("/metrics", async (req, res) => {
  res.set("Content-Type", client.register.contentType);
  res.end(await client.register.metrics());
});
```

### Traces

A trace is a tree of timed **spans** — one per service or significant operation — that records exactly where time was spent across a distributed request. Instrumenting with the OpenTelemetry SDK and viewing in Jaeger or Tempo shows the full call chain: which service was slow, which DB query blocked, and how long each hop took.

**Where time was spent**

```typescript
import { trace } from "@opentelemetry/api";

const tracer = trace.getTracer("my-app");

// Trace function
async function getUserOrders(userId: number) {
  const span = tracer.startSpan("getUserOrders");
  span.setAttribute("user.id", userId);

  try {
    const user = await fetchUser(userId);
    const orders = await fetchOrders(userId);

    span.setStatus({ code: 0 }); // OK
    return { user, orders };
  } catch (err) {
    span.recordException(err);
    span.setStatus({ code: 1, message: err.message }); // Error
    throw err;
  } finally {
    span.end();
  }
}
```

**Trace visualization:**

```
getUserOrders (200ms)
  ├─ fetchUser (50ms)
  │   └─ DB query (40ms)
  └─ fetchOrders (120ms)
      ├─ DB query (80ms)
      └─ Cache lookup (20ms)
```

## 26.2 SLI / SLO / SLA

### SLI (Service Level Indicator)

An SLI is the raw metric you actually measure — usually the *ratio of good events to total events* (e.g., successful requests / total requests). It's the foundation everything else is built on.

**Measurement**: What you actually observe

```
SLI = (successful requests / total requests) * 100

Example: 99.2% of requests in last 30 days completed successfully
```

### SLO (Service Level Objective)

An SLO is the internal reliability target the engineering team commits to — a number *above* the SLA, set by engineering not by contract, that triggers reliability work when breached.

**Target**: What you aim for

```
SLO: 99.5% of requests should complete successfully

Target availability: 99.9% uptime
Target latency: p95 < 200ms
```

### SLA (Service Level Agreement)

An SLA is the formal, customer-facing contract — typically with financial penalties for breach. It should always be *weaker* than your internal SLO, so breaching the SLO is a warning signal well before you breach the SLA.

**Contract**: What you promise customers

```
SLA: 99.9% uptime, or customers get 10% credit

Penalties if SLO not met
```

### Error Budget

The error budget is the amount of unreliability the SLO allows — if your target is 99.9%, you have 0.1% of requests (roughly 43 min/month of downtime) to spend. When it's exhausted, freeze risky changes and invest in reliability until it replenishes.

```
Error Budget = 100% - SLO

SLO = 99.9% → Error Budget = 0.1%
= 43.2 minutes downtime/month

If Error Budget exhausted:
  - Freeze feature releases
  - Focus on reliability
  - Pay down tech debt
```

## 26.3 Alerting

### Good Alerts

Good alerts fire on *symptoms* (users experiencing errors) rather than *causes* (CPU spiked), include context and a runbook link, and are tuned to avoid false positives — alert fatigue from noisy alerts is as dangerous as no alerting, because on-call engineers start ignoring pages.

```yaml
# Prometheus alert rule
groups:
  - name: api-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
          > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "{{ $value | humanizePercentage }} of requests are failing"
```

**Best practices:**

- Alert on symptoms (users affected), not causes
- Include runbook links
- Set appropriate thresholds (avoid alert fatigue)
- Use severity levels (critical, warning, info)

## 26.4 APM and Observability Tooling

You rarely build observability from scratch — you adopt a platform. **APM (Application Performance Monitoring)** tools auto-instrument your app to capture traces, errors, and metrics with minimal code, then visualize and alert on them. Knowing the landscape and the build-vs-buy trade-off is expected at senior level.

- **Datadog** — the market-leading SaaS "single pane of glass": metrics, logs, traces, APM, RUM (real-user monitoring), and security in one product with deep integrations. Excellent UX and correlation; the well-known downside is **cost**, which scales aggressively with hosts, custom metrics, and log volume.
- **New Relic** — a long-standing APM platform similar in scope to Datadog, known for strong code-level transaction tracing; shifted to a consumption (data-ingest) pricing model.
- **Grafana stack (open-source / self-hosted)** — **Prometheus** for metrics (pull-based scraping + PromQL), **Loki** for logs, **Tempo** for traces, all visualized in **Grafana**. The cost-effective, no-lock-in choice favored by Kubernetes shops; the trade-off is you operate it yourself.
- **AWS CloudWatch + X-Ray** — the native AWS option. CloudWatch collects logs/metrics/alarms from AWS services automatically; **X-Ray** provides distributed tracing across Lambda/ECS/API Gateway. Convenient and zero-setup for AWS-native stacks, weaker as a general cross-cloud platform.
- **OpenTelemetry** — not a backend but the **instrumentation standard**: instrument once with OTel SDKs/collector and export to *any* of the above. The strategic choice to avoid vendor lock-in — you can switch from Datadog to Grafana without re-instrumenting code.

**Build vs buy:** SaaS (Datadog/New Relic) trades money for speed and zero operational burden — usually right for small/mid teams who shouldn't be running a metrics cluster. Self-hosted (Grafana stack) trades engineering time for cost control and data sovereignty — right at large scale where SaaS bills explode or where data can't leave your environment. Instrumenting with OpenTelemetry keeps that decision reversible.

## Observability Priority Summary

| Topic                               | Priority     |
| ----------------------------------- | ------------ |
| **Three Pillars**                   |              |
| Logs (structured, correlation ID)   | **Critical** |
| Metrics (RED, USE)                  | **Critical** |
| Traces (spans, distributed tracing) | **Critical** |
| OpenTelemetry (vendor-neutral)      | **Important**|
| **SLI/SLO/SLA**                     |              |
| Understanding + Error Budget        | **Deep**     |
| **Alerting**                        |              |
| Alert on symptoms                   | **Deep**     |
| **APM / Tooling**                   |              |
| Datadog / New Relic (SaaS APM)      | **Important**|
| Grafana/Prometheus/Loki/Tempo       | **Learn**    |
| CloudWatch + X-Ray                  | **Learn**    |
| Build vs buy trade-off              | **Deep**     |

---


# 27. Application Security

Security is a cross-cutting concern that a senior engineer is expected to own, not delegate. The mindset shift from mid to senior is treating every input as hostile and every trust boundary as a place where things can go wrong — the network edge, the database query, the deserializer, the third-party dependency. Two frameworks anchor the conversation: the **OWASP Top 10** (the most common, highest-impact web vulnerabilities) and the **defense-in-depth** principle (no single control is trusted; you layer validation, authn/authz, least privilege, and monitoring so one failure doesn't become a breach). This section covers the testing types that catch issues at different stages, the OWASP Top 10 with concrete fixes, and the HTTP-level hardening you should apply by default.

## 27.1 Security Testing Types

Security testing happens at different points in the lifecycle, and the three types below are complementary, not alternatives — mature pipelines run all of them. The mental model: **SAST** reads your source code before it runs (finds injection patterns, hardcoded secrets), **SCA** audits your dependencies (the majority of a modern app's code is third-party, and that's where most known CVEs live), and **DAST** attacks the running application from the outside like a real attacker would (finds misconfigurations and runtime-only issues SAST can't see).

### SAST (Static Application Security Testing)

SAST analyzes source code *without executing it*, catching injection patterns, hardcoded secrets, dangerous API calls, and code quality issues early — before the code ever runs. It integrates into CI to block merges on security findings.

#### SonarQube / SonarCloud

SonarQube (self-hosted) and SonarCloud (SaaS) are the most widely used SAST platforms in enterprise environments. Beyond pure security, they cover code quality (bugs, code smells, duplication, coverage) in a single tool — making them the practical choice for teams that want one scan to cover both.

Key concepts:

- **Quality Gate** — a configurable pass/fail policy applied to every analysis. The default gate fails if new code introduces any blocker/critical vulnerability, drops below 80% coverage on new code, or has a high duplication ratio. CI integration makes the Quality Gate the merge gate.
- **Security Hotspots** — code patterns that *may* be vulnerabilities but require human review (e.g., a `Math.random()` call that might be used for security-sensitive tokens). Distinct from confirmed vulnerabilities, which block the Quality Gate automatically.
- **PR decoration** — SonarCloud/SonarQube posts findings directly on GitHub/GitLab pull requests as inline comments, surfacing issues without leaving the review workflow.

```properties
# sonar-project.properties  (repo root)
sonar.projectKey=my-org_my-repo
sonar.organization=my-org

# Sources and tests
sonar.sources=src
sonar.tests=src
sonar.test.inclusions=**/*.test.ts,**/*.spec.ts
sonar.exclusions=**/node_modules/**,**/dist/**,**/*.d.ts

# Coverage — point at the LCOV report generated by your test runner
sonar.javascript.lcov.reportPaths=coverage/lcov.info

# Quality Gate behaviour: fail the analysis if the gate is not passed
sonar.qualitygate.wait=true
```

#### CodeQL

GitHub's semantic code analysis engine — it parses code into a queryable database and runs queries against it. Built into **GitHub Advanced Security** (free for public repos). Better than pattern-matching tools at finding complex vulnerability chains (e.g., taint analysis: tracking untrusted input from an HTTP parameter through multiple function calls to a SQL query).

```yaml
# .github/workflows/codeql.yml
- name: Initialize CodeQL
  uses: github/codeql-action/init@v3
  with:
    languages: javascript-typescript
    queries: security-extended  # broader rule set than default

- name: Perform CodeQL Analysis
  uses: github/codeql-action/analyze@v3
```

#### Semgrep

Lightweight, pattern-based SAST with a large open-source rule registry. Fast enough to run on every commit; rules are readable YAML patterns you can write yourself — making it the right tool for enforcing custom coding standards beyond what off-the-shelf tools cover (e.g., "never call `execSync` with a string containing user input").

```yaml
# semgrep rule: detect use of dangerous eval patterns
rules:
  - id: no-eval-user-input
    patterns:
      - pattern: eval($X)
      - pattern-not: eval("...")   # string literal is fine
    message: "eval() with dynamic input is dangerous"
    languages: [javascript, typescript]
    severity: ERROR
```

Run in CI: `semgrep --config=p/javascript --error` (exits non-zero on findings).

**SAST tool comparison:**

| Tool | Strength | Hosting | Best for |
| --- | --- | --- | --- |
| SonarQube/Cloud | Quality + security in one, PR decoration | Self-hosted / SaaS | Enterprise teams, code quality gates |
| CodeQL | Deep taint analysis, semantic queries | GitHub Actions (free) | Complex vuln chains, GitHub shops |
| Semgrep | Custom rules, fast, readable config | CI / SaaS | Enforcing team-specific patterns |

### SCA (Software Composition Analysis)

SCA audits third-party dependencies for known CVEs and license violations. Because the majority of a modern app's code is open-source, this is often the highest-yield security scan — one vulnerable transitive dependency can expose a critical RCE.

#### Snyk

Snyk is the most widely adopted SCA platform for developer-first security. It goes beyond detecting vulnerabilities to actively helping fix them.

Core capabilities:

- **Dependency scanning** — scans `package.json`, `yarn.lock`, `pom.xml`, `requirements.txt`, etc. against Snyk's vulnerability database (larger than the NVD for npm/Python/Java ecosystems).
- **Fix PRs** — when a newer non-vulnerable version is available, Snyk automatically opens a pull request to upgrade the dependency. This is the biggest practical differentiator from `npm audit`.
- **Container scanning** — scans Docker images for OS-level CVEs (base image packages) in addition to app dependencies.
- **IaC scanning** — checks Terraform, Kubernetes manifests, and CloudFormation for misconfigurations.
- **License compliance** — flags copyleft licenses (GPL, AGPL) in commercial products, preventing legal exposure.
- **Policies** — define org-wide rules: which severities block CI, which CVEs to ignore (with expiry), and what license types are prohibited.

```yaml
# .github/workflows/snyk.yml
- name: Run Snyk to check for vulnerabilities
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high  # only fail on high/critical
```

```bash
# Local usage
snyk test                          # scan current project
snyk test --all-projects           # monorepo: scan every package
snyk monitor                       # upload snapshot to Snyk dashboard for ongoing monitoring
snyk container test myapp:latest   # scan Docker image
snyk iac test ./infra/             # scan IaC files
```

**Snyk vs npm audit:** `npm audit` is free and built-in but produces false positives, has no fix PRs, and its vulnerability database lags Snyk's. Snyk is the production choice; `npm audit` is the baseline check in environments without a Snyk license.

#### Trivy

Open-source, zero-config vulnerability scanner from Aqua Security. Scans containers, filesystems, git repos, and IaC — a good lightweight alternative to Snyk for teams that need container scanning without a SaaS dependency.

```bash
trivy image myapp:latest           # scan container image
trivy fs .                         # scan filesystem / project
trivy repo https://github.com/...  # scan remote repo
```

**SCA tool comparison:**

| Tool | Strength | Fix PRs | Container | IaC | Cost |
| --- | --- | --- | --- | --- | --- |
| Snyk | Largest DB, fix PRs, policies | Yes | Yes | Yes | Freemium / paid |
| npm audit | Zero setup, built-in | No | No | No | Free |
| Trivy | OSS, fast, multi-target | No | Yes | Yes | Free |

### DAST (Dynamic Application Security Testing)

DAST attacks a *running* application from the outside — like a real attacker — finding misconfigurations, authentication bypasses, and runtime-only issues that static analysis can't see.

```bash
# OWASP ZAP — baseline scan (passive, safe for CI)
docker run -t owasp/zap2docker-stable zap-baseline.py -t https://myapp.com

# Full active scan (sends attack payloads — use only against test environments)
docker run -t owasp/zap2docker-stable zap-full-scan.py -t https://myapp-staging.com
```

DAST requires a running environment, so it typically runs against a staging deployment in CI rather than on every PR.

## 27.2 OWASP Top 10

### 1. Broken Access Control

The #1 OWASP risk (2021): a user can access data or perform actions they shouldn't — missing authorization checks, insecure direct object references (`/users/:id` without ownership verification), or misconfigured roles are the most common forms.

```typescript
// ❌ Bad: No authorization check
app.delete("/users/:id", async (req, res) => {
  await db.users.delete(req.params.id);
  res.sendStatus(204);
});

// ✅ Good: Check ownership
app.delete("/users/:id", authenticateJWT, async (req, res) => {
  if (req.user.id !== parseInt(req.params.id) && req.user.role !== "admin") {
    return res.status(403).json({ error: "Forbidden" });
  }

  await db.users.delete(req.params.id);
  res.sendStatus(204);
});
```

### 2. Cryptographic Failures

Previously called "Sensitive Data Exposure": storing passwords in plain text or with weak hashes (MD5), transmitting secrets over HTTP, or exposing keys in source control. Use bcrypt/argon2 for passwords, HTTPS everywhere, and rotate secrets out of code into environment variables.

```typescript
// ❌ Bad: Plain text password
await db.users.create({ email, password });

// ✅ Good: Hashed password
import bcrypt from "bcrypt";

const hashedPassword = await bcrypt.hash(password, 10);
await db.users.create({ email, password: hashedPassword });

// Verify
const match = await bcrypt.compare(password, user.password);
```

### 3. Injection

Injection attacks insert hostile data into a query or interpreter — SQL injection is the classic form, but command injection, LDAP injection, and XSS follow the same pattern. The defense in all cases: parameterized queries or an ORM; never interpolate untrusted input directly into a query string.

```typescript
// ❌ Bad: SQL injection
const users = await db.query(`SELECT * FROM users WHERE email = '${email}'`);

// ✅ Good: Parameterized query
const users = await db.query("SELECT * FROM users WHERE email = $1", [email]);

// ✅ Good: ORM
const users = await prisma.user.findMany({ where: { email } });
```

### 4. Insecure Design

A design-level category for flows that are missing necessary security controls by architecture — no rate limiting on a password-reset endpoint, no token expiry, predictable reset tokens. The fix requires rethinking the design, not just patching the code.

```typescript
// ✅ Good: Secure password reset flow
app.post("/reset-password", async (req, res) => {
  const user = await db.users.findByEmail(req.body.email);
  const token = crypto.randomBytes(32).toString("hex");
  await db.resetTokens.create({
    userId: user.id,
    token,
    expiresAt: Date.now() + 3600000,
  });
  await emailService.sendResetLink(user.email, token);
  res.json({ message: "Reset link sent" });
});

app.post("/reset-password/:token", async (req, res) => {
  const resetToken = await db.resetTokens.findByToken(req.params.token);
  if (!resetToken || resetToken.expiresAt < Date.now()) {
    return res.status(400).json({ error: "Invalid or expired token" });
  }

  const hashedPassword = await bcrypt.hash(req.body.password, 10);
  await db.users.update(resetToken.userId, { password: hashedPassword });
  await db.resetTokens.delete(resetToken.id);
  res.json({ message: "Password reset successful" });
});
```

### 5. Security Misconfiguration

Default credentials left in place, stack traces exposed in error responses, debug endpoints open in production, missing security headers — any deviation from a hardened baseline. The fix: environment-specific configs, generic error messages in production, and automated baseline scans.

```typescript
// ✅ Good: Generic error message
app.use((err, req, res, next) => {
  logger.error({ err, req }, "Request error");

  const message =
    process.env.NODE_ENV === "production"
      ? "Internal server error"
      : err.message;

  res.status(err.status || 500).json({ error: message });
});
```

### 7. Identification & Authentication Failures

Weak authentication mechanisms: no rate limiting on login endpoints (enabling brute force), no account lockout, sessions that never expire, or credentials stored client-side. Defense: rate limiting, lockout after N failures, secure session management.

```typescript
// ✅ Good: Rate limiting + account lockout
import rateLimit from "express-rate-limit";

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: "Too many login attempts",
});

app.post("/login", loginLimiter, async (req, res) => {
  const user = await authenticateUser(req.body.email, req.body.password);
  if (!user) {
    await db.loginAttempts.increment(req.body.email);
    const attempts = await db.loginAttempts.get(req.body.email);
    if (attempts >= 5) {
      await db.users.lock(req.body.email, Date.now() + 3600000); // 1 hour
    }
    return res.status(401).json({ error: "Invalid credentials" });
  }

  await db.loginAttempts.reset(req.body.email);
  res.json({ token: generateToken(user) });
});
```

### 10. Server-Side Request Forgery (SSRF)

SSRF tricks the server into making outbound requests to internal infrastructure — cloud metadata endpoints (`169.254.169.254`), internal databases, or other services the attacker can't reach directly from outside the network. Defense: allowlist permitted domains; block private IP ranges.

```typescript
// ✅ Good: Whitelist allowed domains
const ALLOWED_DOMAINS = ["api.example.com", "cdn.example.com"];

app.post("/fetch", async (req, res) => {
  const url = new URL(req.body.url);
  if (!ALLOWED_DOMAINS.includes(url.hostname)) {
    return res.status(400).json({ error: "Invalid domain" });
  }

  const response = await fetch(url.toString());
  res.send(await response.text());
});
```

## 27.3 Security Headers

### Helmet.js

Helmet sets a battery of security-relevant HTTP response headers by default — Content-Security-Policy, HSTS, X-Frame-Options, and more — hardening Express apps against common web attacks with a single `app.use(helmet())` call.

```typescript
import helmet from "helmet";

app.use(
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", "data:", "https:"],
      },
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true,
    },
  }),
);
```

## Security Priority Summary

| Topic                  | Priority     |
| ---------------------- | ------------ |
| **Testing**            |              |
| SAST — SonarQube/Cloud (Quality Gates, PR decoration) | **Deep** |
| SAST — CodeQL (taint analysis, semantic queries) | **Important** |
| SAST — Semgrep (custom rules, CI enforcement) | **Learn** |
| SCA — Snyk (fix PRs, container/IaC scanning, policies) | **Deep** |
| SCA — npm audit / Trivy (baseline / OSS alternatives) | **Important** |
| DAST — OWASP ZAP (baseline vs active scan) | **Know** |
| **OWASP Top 10**       | **Critical** |
| **Security Headers**   |              |
| Helmet.js              | **Deep**     |
| CORS                   | **Deep**     |
| **Input Validation**   |              |
| Zod                    | **Deep**     |

---

---

_End of Part 5. Continue to **Part 6** (Cloud & Data Engineering) in [`06_Cloud_DataEng_28-30.md`](./06_Cloud_DataEng_28-30.md), or return to the [README](./README.md)._
