# Solution Architecture (34-36)
### Distributed Systems, System Design, Solution Architecture

---

# 34. Distributed Systems

Once a system spans more than one machine, a new class of problems appears that simply doesn't exist on a single box: the network can drop or delay messages, individual nodes can fail independently, and there is no longer a single global "now" that every component agrees on. Distributed systems theory is the body of knowledge for reasoning about these failure modes and the trade-offs they force. For a senior engineer, the value isn't memorizing theorems — it's being able to look at a design and immediately ask the right questions: *What happens when this network link fails? Can this operation be safely retried? How stale can this read be?* This section covers the core ideas interviewers probe and that you'll reach for in real system design.

## 34.1 CAP Theorem

The CAP theorem is the starting point for almost every distributed systems discussion. It states that when a network **partition** occurs (some nodes can't talk to others), a system must choose between **consistency** (every read sees the latest write) and **availability** (every request gets a non-error response). You cannot have both during a partition — that's the whole theorem. The common "pick 2 of 3" framing is slightly misleading: partition tolerance isn't optional in any real network, so the genuine choice is **CP vs AP** *during a partition*. When the network is healthy, you get both C and A.

**You can only choose 2 out of 3:**

- **C**onsistency: All nodes see the same data at the same time
- **A**vailability: Every request receives a response
- **P**artition tolerance: System continues despite network partitions

**In practice, partitions WILL happen, so choose:**

- **CP**: Consistency + Partition tolerance (sacrifice availability)
  - Examples: MongoDB, HBase, Redis
  - Use when: Financial transactions, inventory
- **AP**: Availability + Partition tolerance (sacrifice consistency)
  - Examples: Cassandra, DynamoDB
  - Use when: Social media feeds, analytics

## 34.2 Consistency Models

Consistency isn't binary — it's a spectrum of guarantees about *when* a write becomes visible to subsequent reads. Stronger models are easier to reason about but cost latency and availability (they require coordination); weaker models are faster and more available but force the application to tolerate stale or conflicting data. The two endpoints you must be able to contrast in an interview are strong and eventual consistency, with **read-your-own-writes**, **monotonic reads**, and **causal consistency** as useful intermediate points.

### Strong Consistency

Under strong consistency, every read returns the most recently committed write — any replica that acknowledges a write has the latest value. This requires coordination on every write, adding latency and reducing availability under network partitions.

```
Write to A → Read from B → Gets latest value
```

**Example:** Relational databases (ACID transactions)

### Eventual Consistency

Under eventual consistency, a write propagates asynchronously — a read immediately after a write may return a stale value, but given enough time (typically milliseconds to seconds) all replicas converge. It trades read correctness for higher throughput and availability.

```
Write to A → Read from B (immediately) → May get stale value
Wait... → Read from B → Gets latest value
```

**Example:** DynamoDB, Cassandra, DNS

## 34.3 Distributed Transactions

A single-database transaction gives you ACID guarantees for free. The moment a business operation must update two independent services or databases (debit account A in one service, credit account B in another), you've lost that free lunch — there's no shared transaction log to roll back across process boundaries. Distributed transactions are the techniques for keeping multiple systems consistent. The honest senior answer is usually: **avoid them when you can** (model the operation to live in one service), and when you can't, prefer the Saga pattern over 2PC because 2PC's blocking behavior makes it fragile at scale.

### Two-Phase Commit (2PC)

2PC coordinates a commit across multiple participants: all must vote "yes" in Phase 1 before any commit happens in Phase 2. Its critical weakness is **blocking** — if the coordinator crashes after participants vote yes but before Phase 2, they hold locks indefinitely waiting for a decision that never comes.

```
Phase 1: Prepare
  Coordinator → Participants: "Can you commit?"
  Participants → Coordinator: "Yes" or "No"

Phase 2: Commit
  If all "Yes":
    Coordinator → Participants: "Commit"
  Else:
    Coordinator → Participants: "Abort"
```

**Problems:**

- Blocking: If coordinator fails, participants wait forever
- Not fault-tolerant

### Saga Pattern

See section 16.2 for implementation.

**Two approaches:**

1. **Orchestration**: Central coordinator
2. **Choreography**: Services emit events, others react

## 34.4 Idempotency

An operation is **idempotent** if performing it multiple times has the same effect as performing it once. This is one of the most important practical concepts in distributed systems, because networks force you to retry: a client sends a request, the response is lost, and the client can't tell whether the server processed it. If the operation is idempotent, the client can safely retry; if not, the retry causes a double charge, a duplicate order, or a corrupted balance. The standard mechanism is an **idempotency key** — a client-generated unique ID that the server records, so a repeated request with the same key returns the original result instead of re-executing. Note that `GET`, `PUT`, and `DELETE` are idempotent by HTTP semantics, while `POST` is not — which is exactly why payment and order-creation endpoints need explicit idempotency keys.

### Why It Matters

The practical danger of non-idempotent operations is the **retry-induced duplicate**: a network error forces the client to retry, but the server already processed the first request — leading to double charges, duplicate orders, or corrupted balances.

```typescript
// ❌ Non-idempotent: Creates duplicate charges
app.post("/charge", async (req, res) => {
  const charge = await stripe.charges.create({ amount: 100 });
  res.json(charge);
});

// Network error → Client retries → Double charge!

// ✅ Idempotent: Uses idempotency key
app.post("/charge", async (req, res) => {
  const idempotencyKey = req.headers["idempotency-key"];

  // Check if already processed
  const existing = await db.charges.findByKey(idempotencyKey);
  if (existing) {
    return res.json(existing);
  }

  const charge = await stripe.charges.create(
    { amount: 100 },
    { idempotencyKey },
  );

  await db.charges.save({ key: idempotencyKey, charge });
  res.json(charge);
});
```

## 34.5 Distributed Locking

A normal mutex protects a critical section within one process. When multiple processes or servers must coordinate exclusive access to a shared resource (only one worker should process a given order), you need a **distributed lock** — a lock that lives in a shared store all of them can see. The classic implementation uses Redis (Redlock), but distributed locks are deceptively dangerous: a lock holder can pause (GC, network stall) past its TTL while still believing it holds the lock, letting a second holder in. Always set a TTL (so a crashed holder doesn't lock the resource forever), and for correctness-critical work prefer a **fencing token** (a monotonically increasing number checked by the resource) over relying on the lock alone. When you can, design the work to be idempotent so a brief double-execution is harmless — that's more robust than perfect mutual exclusion.

### Redis Lock (Redlock)

Redlock acquires a distributed lock by writing a key to Redis with a TTL — if the lock holder crashes, the TTL ensures automatic expiry so no process waits forever. The `try/finally` pattern guarantees the lock is released even if the protected work throws.

```typescript
import Redlock from "redlock";
import Redis from "ioredis";

const redis = new Redis();
const redlock = new Redlock([redis], {
  retryCount: 10,
  retryDelay: 200,
});

async function processOrder(orderId: number) {
  const lock = await redlock.acquire([`locks:order:${orderId}`], 5000); // 5s TTL

  try {
    // Only one process can execute this at a time
    const order = await db.orders.findById(orderId);

    if (order.status !== "pending") {
      return; // Already processed
    }

    await processPayment(order);
    await updateInventory(order);
    await db.orders.update(orderId, { status: "completed" });
  } finally {
    await lock.release();
  }
}
```

## 34.6 Consistent Hashing

Consistent hashing solves a specific scaling problem: how to distribute keys across a changing set of servers (cache nodes, shards) so that adding or removing a server moves as few keys as possible. With naive `hash(key) % N`, changing `N` remaps almost every key — catastrophic for a cache (mass misses) or a sharded database (mass data movement). Consistent hashing places both servers and keys on a conceptual ring; each key belongs to the next server clockwise, so adding a node only steals keys from its immediate neighbor — roughly `1/N` of the total. **Virtual nodes** (placing each physical server at many ring positions) smooth out the otherwise-uneven distribution. It's the backbone of systems like Cassandra, DynamoDB partitioning, and distributed caches.

### Problem: Traditional Hashing

With naive modulo hashing (`hash(key) % N`), adding or removing a single node changes `N` and remaps the majority of keys — in a cache this triggers a mass-miss storm; in a sharded database it forces a mass data migration.

```typescript
function getServer(key: string, serverCount: number): number {
  return hash(key) % serverCount;
}

// getServer('user:123', 3) → server 1
// Add server → getServer('user:123', 4) → server 2 (moved!)
```

### Solution: Consistent Hashing

Consistent hashing places both servers and keys on a conceptual ring — each key routes to the nearest server clockwise, so adding a server only takes keys from its immediate neighbor (roughly `1/N` of the total). **Virtual nodes** place each physical server at many ring positions to smooth an otherwise-uneven distribution.

```typescript
class ConsistentHash {
  private ring: Map<number, string> = new Map();
  private sortedKeys: number[] = [];
  private vnodes = 150; // Virtual nodes per server

  addServer(server: string) {
    for (let i = 0; i < this.vnodes; i++) {
      const hash = this.hash(`${server}:${i}`);
      this.ring.set(hash, server);
      this.sortedKeys.push(hash);
    }
    this.sortedKeys.sort((a, b) => a - b);
  }

  getServer(key: string): string {
    const hash = this.hash(key);

    // Find first server >= hash
    for (const ringHash of this.sortedKeys) {
      if (ringHash >= hash) {
        return this.ring.get(ringHash)!;
      }
    }

    // Wrap around to first server
    return this.ring.get(this.sortedKeys[0])!;
  }

  private hash(key: string): number {
    let hash = 0;
    for (let i = 0; i < key.length; i++) {
      hash = (hash << 5) - hash + key.charCodeAt(i);
      hash |= 0;
    }
    return Math.abs(hash);
  }
}
```

**Use cases:** Load balancing, sharding, caching

## 34.7 Cache Strategies

Caching trades freshness for speed and load reduction, and the strategy you pick determines *who* writes to the cache and *when*. The dominant pattern is **cache-aside** (the application checks the cache, falls back to the DB on a miss, and populates the cache itself) because it's simple and resilient — a cache outage just means slower reads, not failures. **Write-through** keeps the cache in sync on every write (consistent but slower writes); **write-behind** buffers writes and flushes asynchronously (fast but can lose data on crash). The hard part of caching is always **invalidation** — deciding when cached data is stale — and the two failure modes to name in interviews are **cache stampede** (many requests rebuild the same expired key at once) and **cache penetration** (requests for non-existent keys bypass the cache and hammer the DB).

### Cache-Aside (Lazy Loading)

In cache-aside, the application owns the cache: check the cache first, fall back to the database on a miss, then populate the cache. A cache failure degrades to slower reads, not errors — making it the most resilient pattern and the default choice.

```typescript
async function getUser(id: number) {
  const cached = await redis.get(`user:${id}`);
  if (cached) return JSON.parse(cached);

  const user = await db.users.findById(id);
  await redis.setex(`user:${id}`, 3600, JSON.stringify(user));
  return user;
}
```

### Write-Through

Write-through updates the cache synchronously on every write, keeping it always in sync with the database. Writes are slower, but reads are always fresh — ideal when stale reads are unacceptable and write throughput is not the bottleneck.

```typescript
async function updateUser(id: number, data: any) {
  const user = await db.users.update(id, data);
  await redis.setex(`user:${id}`, 3600, JSON.stringify(user));
  return user;
}
```

### Cache Stampede (Thundering Herd)

A cache stampede occurs when a popular key expires and many concurrent requests simultaneously find a miss — all hit the database at once, amplifying load by orders of magnitude. The fix is a **distributed lock** so only one request rebuilds the cache while the rest wait.

```typescript
// ❌ Problem: Cache expires, 1000 concurrent requests hit DB

// ✅ Solution: Lock (only one request fetches)
import Redlock from "redlock";

async function getPopularItem(id: number) {
  const cached = await redis.get(`item:${id}`);
  if (cached) return JSON.parse(cached);

  const lock = await redlock.acquire([`lock:item:${id}`], 5000);

  try {
    // Check cache again
    const cached = await redis.get(`item:${id}`);
    if (cached) return JSON.parse(cached);

    const item = await db.items.findById(id);
    await redis.setex(`item:${id}`, 3600, JSON.stringify(item));
    return item;
  } finally {
    await lock.release();
  }
}
```

## 34.8 Microservice Architecture

Microservice architecture structures an application as a collection of small, independently deployable services, each owning its own data and communicating via APIs or message brokers. It's a widely used pattern in JS/Node.js — NestJS is largely designed around it, and it's the dominant model for large-scale Node.js backends (alongside serverless, which is a different execution model entirely).

### Monolith vs Microservices

```
Monolith                          Microservices
─────────                         ─────────────
┌──────────────────┐              ┌──────────┐  ┌───────────┐
│  Orders          │              │  Orders  │  │ Inventory │
│  Inventory       │              │ Service  │  │  Service  │
│  Payments        │              └────┬─────┘  └─────┬─────┘
│  Notifications   │                   │  message broker  │
│                  │              ┌────┴─────┐  ┌─────┴─────┐
│  [single DB]     │              │ Payments │  │Notifications│
└──────────────────┘              │ Service  │  │  Service  │
one deployable unit               └──────────┘  └───────────┘
                                  each deploys independently
                                  each owns its own DB
```

| | Monolith | Microservices |
| --- | --- | --- |
| **Deployment** | One unit | Independent per service |
| **Scaling** | Scale everything | Scale individual bottlenecks |
| **Data** | Shared DB | Each service owns its data |
| **Team** | One codebase | Teams own services end-to-end |
| **Latency** | In-process calls | Network calls between services |
| **Complexity** | Low (at first) | High operational overhead |
| **Testing** | Easier to integration test | Requires contract testing |

Start with a monolith. Move to microservices when a specific service has meaningfully different scaling, deployment, or team-ownership requirements — not before.

### Core Concerns

**Service boundaries** — split by business domain (DDD bounded contexts), not by technical layer. `OrderService`, `InventoryService`, `PaymentService` are good boundaries. `DatabaseService` or `ValidationService` are not.

**Inter-service communication** — two modes:
- **Synchronous** (HTTP/REST, gRPC): caller waits for a response. Simpler to reason about but couples availability — if the downstream service is slow or down, the caller is too.
- **Asynchronous** (message broker: RabbitMQ, Kafka, SQS): fire-and-forget or event-driven. Decouples availability but introduces eventual consistency.

**API Gateway** — a single entry point for external clients. Routes requests to the right service, handles cross-cutting concerns (auth, rate limiting, TLS termination) so individual services don't have to.

**Data isolation** — no shared databases between services. If Service A needs data owned by Service B, it asks Service B via API or reads a replicated projection updated via events. Shared databases are the most common way microservices fail to deliver their promises.

**Distributed tracing** — a single user request now spans multiple services. OpenTelemetry trace IDs propagated in headers let you reconstruct the full call chain in Jaeger or Datadog (§26).

### Microservices vs Serverless (Lambda)

This is a common source of confusion because both are "distributed" and "event-driven", but they have fundamentally different execution models.

**Microservices** are long-running processes. A NestJS microservice boots once, maintains persistent connections to its message broker, and processes many requests over its lifetime. RabbitMQ works here because the service maintains a live consumer connection.

**Lambda functions** are stateless and ephemeral. They start on invocation, process one event, and exit. They cannot maintain a persistent connection to RabbitMQ (the process ends before the next message arrives). This is why the AWS event-driven pattern looks different:

```
RabbitMQ/persistent consumer         AWS serverless event-driven
────────────────────────────         ───────────────────────────

[RabbitMQ] ──persistent──► [NestJS   [EventBridge / SQS / SNS]
            connection       Micro-        │
                             service]      │ invokes per event
                                           ▼
                                        [Lambda]  (starts, runs, exits)
```

AWS manages the queue and invokes your Lambda once per message/event — you never write the consumer loop. SQS delivers messages in batches to Lambda; SNS fans out a single event to multiple Lambda subscribers; EventBridge routes events by pattern to different Lambdas. The equivalent of a persistent RabbitMQ consumer is an SQS queue with a Lambda event source mapping.

**Rule of thumb:**
- Long-running service with shared state, persistent broker connections, or steady traffic → containerised microservice
- Event-driven, spiky traffic, short-lived processing, no persistent state needed → Lambda + AWS event services

## Distributed Systems Priority Summary

| Topic                                 | Priority     |
| ------------------------------------- | ------------ |
| CAP Theorem (CP vs AP)                | **Critical** |
| Consistency Models (strong, eventual) | **Deep**     |
| Distributed Transactions (2PC, Saga)  | **Critical** |
| Idempotency (keys, exactly-once)      | **Critical** |
| Distributed Locking (Redlock)         | **Deep**     |
| Consistent Hashing                    | **Deep**     |
| Cache strategies + invalidation       | **Critical** |
| Cache stampede                        | **Know**     |
| **Microservice Architecture**         |              |
| Monolith vs microservices trade-offs  | **Critical** |
| Service boundaries & data isolation   | **Deep**     |
| Microservices vs Lambda/serverless    | **Deep**     |

---


# 35. System Design

The system design interview tests whether you can take an ambiguous, open-ended prompt ("design Twitter") and drive it to a concrete, defensible architecture while reasoning out loud about trade-offs. Interviewers are not looking for the one right answer — there isn't one — they're evaluating your *process*: do you clarify requirements before designing, do you estimate scale to justify decisions, do you identify bottlenecks, and can you articulate why you chose SQL over NoSQL or fan-out-on-write over fan-out-on-read? The single biggest mistake is jumping to a solution before nailing down requirements and scale. The method below keeps you structured under pressure; the practice problems train the patterns that recur across most prompts (caching, sharding, queues, fan-out).

## 35.1 Method

### 1. Clarify Requirements (5 min)

Before drawing a single box, pin down the functional scope and non-functional constraints. Getting these wrong early means designing the wrong system — an interviewer who hears you clarifying scale and consistency requirements signals that you know what drives architecture decisions.

**Functional:**

- What features? (user signup, post creation, feed)
- Scale? (100K users, 1M posts/day)

**Non-functional:**

- Latency? (p95 < 200ms)
- Availability? (99.9%)
- Consistency? (eventual OK? strong required?)

### 2. Back-of-Envelope Estimation (5 min)

Order-of-magnitude estimates anchor every subsequent design decision: request rate determines whether you need a single server or a fleet; storage volume determines whether a single database is viable or sharding is required.

```
Users: 100M
DAU: 10M (10%)
Requests/day: 10M × 10 requests = 100M
Requests/sec: 100M / 86400 ≈ 1,200 RPS
Peak (3x): 3,600 RPS

Storage:
  - Post: 1 KB
  - Posts/day: 1M
  - Storage/day: 1 GB
  - Storage/year: 365 GB
  - 5 years: 1.8 TB (3.6 TB with replication)
```

### 3. High-Level Design (10 min)

Sketch the major components and their relationships — enough to show data flow from client to database and back. This is the surface you'll drill into, so resist adding detail before agreeing on the overall shape.

```
Client → Load Balancer → API Servers → Database
                                      → Cache
```

### 4. Deep Dive (20 min)

Pick the two or three areas where real complexity lives — usually the database schema, caching layer, and whatever bottleneck the estimation revealed — and work through them in detail with trade-off reasoning.

- Database schema
- API design
- Caching strategy
- Bottlenecks

### 5. Trade-offs (5 min)

Close by naming the key trade-offs you made and the alternatives you rejected. Interviewers want to see you understand the cost of every choice; acknowledging what you're sacrificing signals more maturity than claiming your design is the one right answer.

- SQL vs NoSQL
- Consistency vs Availability
- Sync vs Async

## 35.2 Practice Problems

### URL Shortener

The canonical beginner system design problem — covers base-62 encoding for short codes, a counter-vs-hash key generation strategy, a cache-heavy read path (99% reads, 1% writes), and the 302 vs 301 redirect trade-off (302 allows analytics; 301 is cached by browsers).

```
POST /api/shorten
  { url: "https://example.com/very/long/url" }
→ { shortUrl: "https://short.ly/abc123" }

GET /abc123 → 302 Redirect to original URL

Base62 encoding:
  const CHARS = '0-9a-zA-Z';
  function encode(num) {
    let result = '';
    while (num > 0) {
      result = CHARS[num % 62] + result;
      num = Math.floor(num / 62);
    }
    return result;
  }

Database:
  id (auto-increment) | short_code | original_url | created_at

Cache (Redis): short_code → original_url (TTL 1 day)
```

### Chat System (WhatsApp)

A chat system's design hinges on the real-time delivery mechanism (WebSockets for persistent connections), message ordering guarantees (timestamp + ID), online presence detection (heartbeat + Redis), and storage layout for efficient retrieval of message history.

```
WebSocket connection for real-time

Send message:
  Client A → WebSocket → API Server → Message Queue → DB
                                    → WebSocket → Client B

Schema:
  messages: id, sender_id, recipient_id, content, timestamp
  users: id, username, last_seen

Ordering: Use timestamp + message_id

Online presence:
  - Heartbeat every 30s
  - Redis Sorted Set: user_id → last_seen
```

### News Feed (Twitter)

The defining trade-off in news feed design is *fan-out strategy*: fan-out-on-write (precompute feeds at post time — fast reads, expensive writes for high-follower accounts) vs fan-out-on-read (compute at read time — fast writes, slower reads). The hybrid routes high-follower celebrities to read-time fan-out.

```
Two approaches:

1. Fan-out on write (precompute feed)
   - On post: Write to all followers' feeds
   - Pros: Fast read
   - Cons: Slow write for celebrities

2. Fan-out on read (compute on request)
   - On read: Fetch posts from followed users
   - Pros: Fast write
   - Cons: Slow read

Hybrid:
  - Regular users: Fan-out on write
  - Celebrities: Fan-out on read

Schema:
  posts: id, user_id, content, created_at
  follows: follower_id, followee_id
  feeds: user_id, post_id (sorted by created_at)
```

### Rate Limiter

Rate limiting is fundamentally a counting algorithm (token bucket, leaky bucket, sliding window) implemented atomically in Redis with a Lua script to prevent race conditions between the check and the decrement.

```
Token Bucket Algorithm:
  - Bucket capacity: 100 tokens
  - Refill rate: 100 tokens/minute
  - Request consumes 1 token
  - If bucket empty, reject

Redis:
  key: user:123:tokens
  value: { tokens: 100, lastRefill: timestamp }

Lua script (atomic):
  local tokens = redis.call('HGET', key, 'tokens')
  local elapsed = now - lastRefill
  tokens = min(capacity, tokens + elapsed * rate)

  if tokens >= 1 then
    tokens = tokens - 1
    redis.call('HMSET', key, 'tokens', tokens, 'lastRefill', now)
    return 1  -- Allowed
  else
    return 0  -- Rate limited
  end
```

### E-commerce (Inventory)

Inventory management is a concurrency problem: two users simultaneously buying the last item must result in exactly one success. Cover the three standard mechanisms — pessimistic locking (SELECT FOR UPDATE), optimistic locking (version field + conditional update), and Redis atomic decrement.

```
Problem: Race condition (overselling)

Solutions:

1. Pessimistic Locking (SQL)
  BEGIN;
  SELECT * FROM products WHERE id = 1 FOR UPDATE;
  UPDATE products SET stock = stock - 1 WHERE id = 1;
  COMMIT;

2. Optimistic Locking (Version field)
  UPDATE products SET stock = stock - 1, version = version + 1
  WHERE id = 1 AND version = :currentVersion;

3. Redis (Atomic)
  DECR inventory:product:1
  if result < 0:
    INCR inventory:product:1  # Rollback
    reject
```

## 35.3 Key Latencies

Internalizing order-of-magnitude latency differences lets you estimate performance budgets and justify design decisions instantly — it's why a cache reduces latency from milliseconds to microseconds, and why a cross-continent call can't be in a hot loop.

| Operation        | Latency |
| ---------------- | ------- |
| L1 cache         | 0.5 ns  |
| RAM              | 100 ns  |
| SSD read         | 150 μs  |
| Same-DC network  | 0.5 ms  |
| CA → Netherlands | 150 ms  |

## System Design Priority Summary

| Topic                                                       | Priority     |
| ----------------------------------------------------------- | ------------ |
| Method (requirements → estimation → design)                 | **Critical** |
| Practice problems (URL shortener, chat, feed, rate limiter) | **Critical** |
| Trade-offs (SQL/NoSQL, consistency/availability)            | **Deep**     |
| Back-of-envelope calculations                               | **Deep**     |

---
# 36. Solution Architecture

Where §35 teaches the *interview method* for designing a system under time pressure, this section is the architecture *discipline* behind it: the vocabulary of quality attributes (the "-ilities"), the trade-offs between them, and the decision-making and stakeholder practices that turn a design into a defensible, recorded choice. The defining truth of architecture is that **there is no best design, only the best trade-off for this context** — you cannot maximize reliability, scalability, maintainability, *and* cost simultaneously, so the architect's job is to make the trade-offs explicit, align stakeholders on them, and record the reasoning. Interviewers probe this with questions like *"how would you decide between two valid designs?"* and *"what does 'scalable' actually mean here?"* — they want to hear you reason about non-functional requirements, not just draw boxes. This section covers quality attributes, their tensions, the design/decision process, and the alignment tools (RACI) that make it real.

## 36.1 Quality Attributes / NFRs

**Functional requirements** say *what* the system does; **non-functional requirements (NFRs)**, a.k.a. **quality attributes** or the **"-ilities,"** say *how well* it does it. They are what architecture is really about. The big ones:

- **Reliability** — does it do the right thing correctly, even when components fail? (Often quantified as MTBF, error rate.) "Continues to work *correctly*."
- **Availability** — is it *up* and reachable when needed? Quantified in **nines**. A system can be available but unreliable (up, but returning wrong answers) — the two are distinct.
- **Scalability** — can it handle growth in load (users, data, throughput) by adding resources? **Vertical** (bigger box) vs **horizontal** (more boxes).
- **Maintainability** — how easily can engineers understand, change, and extend it? (Modularity, low coupling, observability, tests.) The attribute that quietly dominates total cost of ownership.
- **Performance** — latency and throughput under load.
- **Security**, **Observability**, **Portability**, **Cost** round out the common set.

**The nines** (see §26.2 for SLO context):

| Availability | Downtime/year | Downtime/month |
| --- | --- | --- |
| 99% ("two nines") | 3.65 days | ~7.2 hours |
| 99.9% ("three nines") | 8.77 hours | ~43 min |
| 99.99% ("four nines") | 52.6 min | ~4.3 min |
| 99.999% ("five nines") | 5.26 min | ~26 sec |

Each extra nine is exponentially more expensive — a key trade-off input.

## 36.2 Tradeoffs Between Attributes

You optimize one attribute at the expense of others; naming the tension is the senior skill.

- **Availability vs Consistency** — the **CAP theorem** (§34.1): under a network partition you must choose. A shopping cart picks availability (eventual consistency); a bank ledger picks consistency. *State which you're choosing and why.*
- **Scalability vs Simplicity/Maintainability** — sharding, caching layers, and microservices buy scale but add operational and cognitive complexity. A monolith is more maintainable until it isn't scalable; premature distribution is a classic mistake (you inherit distributed-systems failure modes for load you don't have).
- **Reliability/Availability vs Cost** — every extra nine means redundancy, multi-region, more on-call. Match the target to the *business need* — a five-nines internal admin tool is waste.
- **Performance vs Maintainability** — hand-tuned, cache-heavy, denormalized code is fast and harder to change. Optimize where measurement says it matters, not everywhere.
- **Latency vs Durability/Consistency** — synchronous replication is safe but slow; async is fast but risks data loss on failover.

**ATAM** (Architecture Tradeoff Analysis Method) is the formal name for systematically evaluating an architecture against prioritized quality attributes and finding the *sensitivity points* (decisions that strongly affect an attribute) and *trade-off points* (decisions affecting several attributes at once). You don't need the full ceremony in an interview, but referencing the *concept* — "I'd identify which decisions are trade-off points across reliability and cost" — signals depth.

**Interview framing:** "How do you choose between two valid architectures?" — Enumerate the relevant quality attributes, rank them *by business priority* (a fintech ranks consistency/auditability; a media site ranks availability/latency), then show which design wins on the top-ranked attributes and what you're consciously sacrificing. The explicit ranking is the answer they want.

## 36.3 Designing Systems & Decision-Making

A repeatable architecture process (complementary to §35's interview steps):

```text
1. Requirements   — functional + NFRs (which -ilities matter, ranked?)
2. Constraints    — budget, team skills, deadlines, existing stack, compliance
3. Drivers        — the few quality attributes that dominate this system
4. Candidates     — 2-3 viable architectures (don't anchor on one)
5. Evaluate       — score each against the ranked attributes & constraints
6. Decide & record — choose, then write an ADR (§33.2) capturing the why
```

**Key principles in modern practice:**

- **Evolutionary architecture** — design for change, because requirements will. Use **fitness functions** (automated checks — e.g., a test that fails if a layer imports a forbidden dependency, or a latency budget enforced in CI) to protect important attributes as the system evolves.
- **Strangler-fig migration** — replace a legacy system incrementally, routing slices of traffic to the new one, rather than a risky big-bang rewrite (ties to the "no rewrite" point in §31.1).
- **Build vs buy** — don't build undifferentiated heavy lifting (auth, payments, search) when a mature service exists; spend complexity budget on your actual differentiator.
- **Code-level patterns** (SOLID, GoF, hexagonal) live in §20 — solution architecture operates a level up, at system/service boundaries.

## 36.4 RACI & Stakeholder Alignment

Architecture is a *social* activity as much as a technical one — a great design that stakeholders don't buy into doesn't ship. **RACI** is a responsibility-assignment matrix that removes ambiguity about *who does what* on a decision or deliverable:

- **R — Responsible** — does the work.
- **A — Accountable** — owns the outcome, final decision-maker. **Exactly one** per item (the rule people most often break).
- **C — Consulted** — gives input *before* the decision (two-way).
- **I — Informed** — told *after* the decision (one-way).

| Activity | Architect | Tech Lead | PO | Security |
| --- | --- | --- | --- | --- |
| Choose datastore | A | R | C | C |
| Define API contract | C | A/R | C | I |
| Approve data retention | C | R | A | C |

**When to use it:** cross-team initiatives, architecture governance, anything where "I thought *you* owned that" is a risk. The discipline of forcing exactly one **A** per row is where most of the value comes from. **Architecture governance** (review boards, ADR review, standards) is the lightweight structure that keeps many teams' decisions coherent without becoming a bottleneck — the modern lean version pushes decisions to teams and uses ADRs + principles rather than a gatekeeping committee.

## 36.5 Architecture Principles

Durable heuristics that guide decisions when no rule fits:

- **Design for failure** — assume every component and network call will fail; build in timeouts, retries with backoff, circuit breakers, graceful degradation (§34 patterns). "Everything fails all the time" (Werner Vogels).
- **Loose coupling, high cohesion** — minimize what each part needs to know about others; changes stay local. The system-level echo of SOLID.
- **KISS / YAGNI at the system level** — don't add a message queue, a cache tier, or a second service for scale you don't have. Complexity is a cost paid forever.
- **Make the reversible decisions fast and the irreversible ones carefully** (Bezos's "one-way vs two-way doors") — don't spend a week of ADR debate on something you can change in an afternoon.
- **Well-Architected pillars** (AWS framing, broadly applicable): operational excellence, security, reliability, performance efficiency, cost optimization, sustainability — a useful checklist for reviewing any design.

**Interview framing:** "What principles guide your architecture?" — Lead with *design for failure* and *match complexity to actual requirements* (YAGNI), then the reversible/irreversible-decision lens. These show maturity: you optimize for change and avoid over-engineering, which is what distinguishes architects from box-drawers.

## Solution Architecture Priority Summary

| Topic | Priority |
| --- | --- |
| Quality attributes / NFRs (the -ilities) | **Critical** |
| Availability vs reliability distinction | **Important** |
| The nines table | **Important** (see §26.2) |
| Trade-offs (avail vs consistency, scale vs simplicity) | **Critical** |
| Ranking attributes by business priority | **Deep** |
| ATAM (concept) | **Know** |
| Design/decision process → ADR | **Important** |
| Evolutionary architecture / fitness functions | **Learn** |
| Strangler-fig migration; build vs buy | **Important** |
| RACI (exactly one Accountable) | **Important** |
| Architecture principles (design for failure, YAGNI) | **Deep** |

---

---

---

_End of Part 8. Continue to **Part 9** (Generative AI) in [`09_GenAI_37.md`](./09_GenAI_37.md), or return to the [README](./README.md)._
