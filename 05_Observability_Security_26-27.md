# Observability & Security (26-27)
### Observability, APM, Application Security

---

# 26. Observability

Monitoring tells you *whether* a system is healthy; **observability** is the property of being able to ask *why* it's behaving the way it is — including questions you didn't anticipate when you instrumented it. The distinction matters at senior level: a dashboard of pre-defined metrics catches known failure modes, but novel production incidents ("checkout is slow, but only for users in EU on mobile") require the ability to slice and correlate rich telemetry after the fact. The foundation is the **three pillars** — logs, metrics, and traces — increasingly unified under **OpenTelemetry (OTel)**, a vendor-neutral standard for generating and exporting telemetry so you're not locked into one backend. Above the raw data sit the practices that make it actionable: SLI/SLO/error budgets (how you define and defend reliability) and alerting (how you get woken up for the right reasons).

## 26.1 Three Pillars

The three pillars answer different questions and are strongest in combination. **Logs** are discrete, timestamped records of events — the detailed "what happened" you read when debugging a specific request. **Metrics** are numeric aggregates over time (request rate, error count, latency percentiles) — cheap to store and ideal for dashboards and alerts, but they tell you *that* something is wrong, not *which* request. **Traces** follow a single request across service boundaries, breaking it into timed spans — they tell you *where* the time went in a distributed call chain. The senior move is correlating them: a metric alert fires, you pivot to the traces for the slow requests, then drill into the logs for one trace via a shared correlation/trace ID.

### Logs

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

**Measurement**: What you actually observe

```
SLI = (successful requests / total requests) * 100

Example: 99.2% of requests in last 30 days completed successfully
```

### SLO (Service Level Objective)

**Target**: What you aim for

```
SLO: 99.5% of requests should complete successfully

Target availability: 99.9% uptime
Target latency: p95 < 200ms
```

### SLA (Service Level Agreement)

**Contract**: What you promise customers

```
SLA: 99.9% uptime, or customers get 10% credit

Penalties if SLO not met
```

### Error Budget

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

Security testing happens at different points in the lifecycle, and the three acronyms below are complementary, not alternatives — mature pipelines run all of them. The mental model: **SAST** reads your source code before it runs (finds injection patterns, hardcoded secrets), **SCA** audits your dependencies (the majority of a modern app's code is third-party, and that's where most known CVEs live), and **DAST** attacks the running application from the outside like a real attacker would (finds misconfigurations and runtime-only issues SAST can't see).

### SAST (Static Application Security Testing)

**Tests:** Your code (before running)

```yaml
# .github/workflows/sast.yml
- name: Run CodeQL
  uses: github/codeql-action/init@v2
  with:
    languages: javascript

- name: Perform CodeQL Analysis
  uses: github/codeql-action/analyze@v2
```

**Tools:** SonarQube, CodeQL, Semgrep

### SCA (Software Composition Analysis)

**Tests:** Dependencies (vulnerabilities, licenses)

```bash
# npm audit
npm audit

# Snyk
snyk test

# Trivy (containers)
trivy image myapp:latest
```

### DAST (Dynamic Application Security Testing)

**Tests:** Running application (black-box)

```bash
# OWASP ZAP
docker run -t owasp/zap2docker-stable zap-baseline.py   -t https://myapp.com
```

## 27.2 OWASP Top 10

### 1. Broken Access Control

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

```typescript
// ❌ Bad: SQL injection
const users = await db.query(`SELECT * FROM users WHERE email = '${email}'`);

// ✅ Good: Parameterized query
const users = await db.query("SELECT * FROM users WHERE email = $1", [email]);

// ✅ Good: ORM
const users = await prisma.user.findMany({ where: { email } });
```

### 4. Insecure Design

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
| SAST (CodeQL, Semgrep) | **Learn**    |
| SCA (npm audit, Snyk)  | **Deep**     |
| DAST (OWASP ZAP)       | **Know**     |
| **OWASP Top 10**       | **Critical** |
| **Security Headers**   |              |
| Helmet.js              | **Deep**     |
| CORS                   | **Deep**     |
| **Input Validation**   |              |
| Zod                    | **Deep**     |

---

---

_End of Part 5. Continue to **Part 6** (Cloud & Data Engineering) in [`06_Cloud_DataEng_28-30.md`](./06_Cloud_DataEng_28-30.md), or return to the [README](./README.md)._
