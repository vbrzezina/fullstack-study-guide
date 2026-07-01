# Backend (16-23)
### Node.js/Express/NestJS, API Design, Authentication, Databases, Design Principles, Messaging & Event Streaming, Data Structures & Algorithms, Real-Time Systems

---

# 16. Node.js, Express, NestJS

Node.js, Express, and NestJS form the dominant JavaScript backend stack. Node runs JavaScript on a single-threaded event loop powered by libuv — a design that makes it excellent for I/O-intensive workloads (many simultaneous network connections, database queries, file reads) but ill-suited for CPU-bound work like image processing or heavy cryptography. Express is the minimal, unopinionated layer on top: a middleware chain and nothing else. NestJS adds structure — Angular-inspired modules, dependency injection, and four lifecycle hooks (Guards, Interceptors, Pipes, Filters) that make large applications testable and consistent. Interviewers probe the event loop deeply because it explains Node's performance characteristics and determines how you reason about concurrency; they probe NestJS lifecycle because it's how production Node apps handle cross-cutting concerns like auth, validation, and logging. This section covers the mechanics, patterns, and production practices you'll reach for in real applications.

## 16.1 Node.js Core

Node's performance story begins and ends with the event loop. A single-threaded process can handle thousands of concurrent connections because *waiting* — for a file, a database query, a network response — doesn't block the thread; it registers a callback and yields control. Understanding the phases of the loop, the order in which callbacks fire, and the difference between microtasks and macrotasks is essential for explaining why Node performs the way it does and for diagnosing ordering bugs where callbacks fire in an unexpected sequence.

### Event Loop Deep Dive

The event loop runs phases in a fixed order — each phase has its own callback queue. Understanding which callbacks land in which phase, and when microtasks (`Promise` callbacks, `queueMicrotask`) run relative to phases, is essential for reasoning about async ordering bugs.

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout, setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  I/O callbacks (TCP errors, etc.)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  Internal
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  I/O, incoming connections
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  setImmediate
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │      close callbacks      │  socket.on('close', ...)
│  └───────────────────────────┘
└────────────────────────────────

Between each phase:
  1. process.nextTick() queue
  2. Microtask queue (Promises)
```

**Key points:**

- `setImmediate` runs in check phase
- `setTimeout(..., 0)` runs in timers phase
- `process.nextTick` runs BEFORE next phase
- Promises are microtasks (run before next phase)

```javascript
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));
process.nextTick(() => console.log("nextTick"));
Promise.resolve().then(() => console.log("promise"));

// Output: nextTick, promise, timeout/immediate (order varies)
```

### Streams & Backpressure

Node's stream API is how you process data that doesn't fit in memory, or data that arrives incrementally — file reads, HTTP responses, database result cursors. A stream is a sequence of chunks rather than a complete buffer: data flows through in pieces and you process each piece as it arrives. **Backpressure** is the mechanism that prevents a fast producer from overwhelming a slow consumer: when the consumer's internal buffer fills, the stream automatically pauses the producer until the buffer drains. The `pipeline` utility (preferred over `.pipe()` with manual error handlers) propagates errors from every stage and cleans up correctly if any step fails.

```typescript
import { createReadStream, createWriteStream } from "fs";
import { pipeline } from "stream/promises";
import { createGzip } from "zlib";

// ❌ Bad: Loads entire file into memory
const data = await fs.readFile("large-file.txt");
await fs.writeFile("output.txt", data);

// ✅ Good: Streams, constant memory
await pipeline(
  createReadStream("large-file.txt"),
  createGzip(),
  createWriteStream("output.txt.gz"),
);

// Custom stream
import { Transform } from "stream";

const upperCaseTransform = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  },
});

await pipeline(
  createReadStream("input.txt"),
  upperCaseTransform,
  createWriteStream("output.txt"),
);
```

**Backpressure:** Stream automatically pauses when consumer can't keep up.

### Worker Threads

Node's event loop is single-threaded: CPU-bound work blocks the loop and delays every other in-flight request. Worker Threads spawn real OS threads that each run their own V8 instance and event loop, communicating with the parent via structured-clone message passing. Unlike child processes, workers can share memory via `SharedArrayBuffer` — large binary payloads (images, video frames) can be transferred without copying. Use workers for computationally expensive operations — bcrypt hashing, image resizing, PDF generation, heavy data transformation — that would otherwise stall the main loop and inflate response latency for unrelated requests.

```typescript
import { Worker } from "worker_threads";

// main.ts
const worker = new Worker("./worker.js", {
  workerData: { input: [1, 2, 3, 4, 5] },
});

worker.on("message", (result) => {
  console.log("Result:", result);
});

worker.on("error", (err) => {
  console.error(err);
});

// worker.js
import { parentPort, workerData } from "worker_threads";

const result = workerData.input.reduce((sum, n) => sum + n, 0);
parentPort.postMessage(result);
```

**Use for:** CPU-intensive tasks (image processing, encryption, compression).

### Node 20/22 Features

Recent Node releases have absorbed several tools that previously required external packages, reducing dependency chain risk and cold-start size. The built-in test runner removes the need for Jest or Mocha in simple test suites; native `.env` support removes `dotenv`; native `fetch` removes `node-fetch`. These are "know they exist" features for interviews, and practical for greenfield projects where you want a minimal dependency footprint.

#### Built-in Test Runner (Node 18+)

```typescript
import { test, describe } from "node:test";
import assert from "node:assert";

describe("Math", () => {
  test("addition", () => {
    assert.strictEqual(1 + 1, 2);
  });

  test("async operation", async () => {
    const result = await fetchData();
    assert.ok(result);
  });
});
```

```bash
node --test
```

#### Native .env Support (Node 20.6+)

```bash
node --env-file=.env app.js
```

#### Native fetch (Node 18+)

```typescript
const response = await fetch("https://api.example.com/data");
const data = await response.json();
```

### ESM vs CommonJS

The JavaScript ecosystem is mid-migration from CommonJS (`require`/`module.exports`) to ECMAScript Modules (`import`/`export`). CJS loads synchronously at runtime; ESM is statically analysable at parse time, which enables tree-shaking, top-level `await`, and better tooling. The interop story is asymmetric: ESM can import CJS (with caveats around default exports), but CJS can only import ESM via `await import()` — dynamic and async. Set `"type": "module"` in `package.json` to make `.js` files ESM; use `.cjs` / `.mjs` extensions to mix both in the same package.

```typescript
// CommonJS
const express = require("express");
module.exports = { app };

// ESM
import express from "express";
export { app };
```

**package.json:**

```json
{
  "type": "module" // Treat .js as ESM
}
```

**Interop:**

```typescript
// Import CJS in ESM
import pkg from "cjs-package";

// Import ESM in CJS (async)
const esmModule = await import("esm-package");
```

### Performance Profiling

Profiling moves you from "this endpoint is slow" to "this specific function consumes 80% of CPU time." The `--inspect` flag enables V8's inspector protocol, which Chrome DevTools connects to for CPU profiles, heap snapshots, and allocation timelines. clinic.js adds higher-level diagnostics: `doctor` identifies event loop lag and I/O stalls; `bubbleprof` visualises async activity as a bubble chart to find what's blocking; `flame` generates CPU flame graphs. Profiling under production load adds roughly 10–15% overhead — prefer staging environments or short, targeted sampling windows rather than running profilers continuously.

#### --inspect (Chrome DevTools)

```bash
node --inspect app.js
# Open chrome://inspect
```

#### clinic.js

```bash
npm install -g clinic

clinic doctor -- node app.js  # Overall health
clinic bubbleprof -- node app.js  # Async operations
clinic flame -- node app.js  # CPU profiling
```

#### 0x (Flamegraphs)

```bash
npm install -g 0x
0x app.js
```

### Graceful Shutdown

When Kubernetes terminates a pod (rolling deploy, scale-down, eviction), it sends `SIGTERM` and waits for the process to exit cleanly — typically a 30-second grace period. Without graceful shutdown, active connections are killed mid-request, leaving clients with broken responses and potentially leaving database transactions in a partial state. The correct sequence: stop accepting new connections (`server.close()`), let existing connections drain, then close database pools and message-broker connections before exiting. Graceful shutdown is mandatory for zero-downtime deployments and is a common senior interview question.

```typescript
import express from "express";
import { createServer } from "http";

const app = express();
const server = createServer(app);

// Track active connections
const connections = new Set();

server.on("connection", (conn) => {
  connections.add(conn);
  conn.on("close", () => connections.delete(conn));
});

async function gracefulShutdown(signal: string) {
  console.log(`Received ${signal}, starting graceful shutdown...`);

  // Stop accepting new connections
  server.close(() => {
    console.log("HTTP server closed");
  });

  // Close active connections
  for (const conn of connections) {
    conn.end();
  }

  // Close database connections, etc.
  await db.close();
  await redis.quit();

  console.log("Graceful shutdown complete");
  process.exit(0);
}

process.on("SIGTERM", () => gracefulShutdown("SIGTERM"));
process.on("SIGINT", () => gracefulShutdown("SIGINT"));

server.listen(3000);
```

### Error Handling

Node has two classes of unhandled errors: `uncaughtException` (synchronous throws that escape all try-catches) and `unhandledRejection` (Promises that reject without a `.catch()`). The correct strategy for both is the same: log the error for observability, then exit with a non-zero code — a clean crash is safer than running in an undefined state. A process manager (PM2, systemd, Kubernetes) restarts the process. Within Express, `express-async-errors` patches route handlers to propagate thrown errors to your four-parameter error middleware automatically, eliminating the need for try-catch wrappers in every async route handler.

```typescript
// Uncaught exceptions
process.on("uncaughtException", (err) => {
  console.error("Uncaught exception:", err);
  process.exit(1); // Crash, let orchestrator restart
});

// Unhandled promise rejections
process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled rejection at:", promise, "reason:", reason);
  process.exit(1);
});

// Async error handling
import "express-async-errors"; // Catches async errors in Express

app.get("/users", async (req, res) => {
  const users = await db.users.findAll(); // Auto-caught if error
  res.json(users);
});

// Error middleware (must have 4 parameters)
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: "Internal server error" });
});
```

## 16.2 Express

Express is the minimal Node.js web framework — a thin layer over Node's `http` module that adds middleware composition, routing, and error handling. Its central concept is the **middleware chain**: a pipeline of functions that each receive `(req, res, next)`, do their work, and either call `next()` to continue, call `next(err)` to jump to error middleware, or end the response. The order middleware is registered determines execution order — security headers and CORS must come before routing, and error middleware (the four-parameter `(err, req, res, next)` form) must be the last handler registered.

### Middleware Chain

The stack executes in registration order. Each function can modify `req`/`res`, call `next()` to continue, call `next(err)` to jump to error middleware, or end the response directly.

```typescript
import express from "express";
import helmet from "helmet";
import cors from "cors";
import rateLimit from "express-rate-limit";

const app = express();

// Order matters!
app.use(helmet()); // Security headers
app.use(cors()); // CORS
app.use(express.json()); // Parse JSON body

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per windowMs
});
app.use(limiter);

// Custom middleware
app.use((req, res, next) => {
  req.requestTime = Date.now();
  next();
});

// Route handlers
app.get("/users", (req, res) => {
  res.json({ users: [] });
});

// Error middleware (must be last)
app.use((err, req, res, next) => {
  console.error(err);
  res.status(err.status || 500).json({ error: err.message });
});
```

### Validation (Zod)

Zod is the standard for runtime schema validation in TypeScript backends because it serves two purposes simultaneously: it validates the shape and constraints of incoming data, and it infers the TypeScript type from the schema definition — so you don't maintain a separate interface alongside your validation rules. `safeParse` returns a discriminated union (success + typed data, or failure + structured error list) without throwing, making it straightforward to return 400 responses. Schemas are composable: `createUserSchema.omit({ id: true })` or `.partial()` reuses base definitions for create vs update operations.

```typescript
import { z } from "zod";

const userSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
  age: z.number().int().min(18).optional(),
});

app.post("/users", async (req, res) => {
  const result = userSchema.safeParse(req.body);

  if (!result.success) {
    return res.status(400).json({ errors: result.error.errors });
  }

  const user = await db.users.create(result.data);
  res.status(201).json(user);
});
```

### Security Best Practices

Express has no built-in security middleware — you opt in explicitly. Three layers every Express API should add: `helmet` sets defensive HTTP response headers (Content-Security-Policy, X-Frame-Options, HSTS) in a single line, closing common browser-side attack vectors; `cors` controls which origins can send cross-origin requests — misconfigured CORS either blocks legitimate frontends or opens the API to unauthorized origins; `express-rate-limit` guards against brute-force and enumeration attacks at the application layer, and should be backed by a Redis store so limits are enforced consistently across all app instances rather than per-process.

#### Helmet.js

```typescript
import helmet from "helmet";

app.use(
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        scriptSrc: ["'self'"],
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

#### CORS

```typescript
import cors from "cors";

app.use(
  cors({
    origin: process.env.ALLOWED_ORIGINS?.split(",") || "*",
    credentials: true,
    methods: ["GET", "POST", "PUT", "DELETE"],
    allowedHeaders: ["Content-Type", "Authorization"],
  }),
);
```

#### Rate Limiting

```typescript
import rateLimit from "express-rate-limit";
import RedisStore from "rate-limit-redis";
import { createClient } from "redis";

const redisClient = createClient();

const limiter = rateLimit({
  store: new RedisStore({
    client: redisClient,
    prefix: "rl:",
  }),
  windowMs: 15 * 60 * 1000,
  max: 100,
  message: "Too many requests, please try again later",
  standardHeaders: true,
  legacyHeaders: false,
});

app.use("/api/", limiter);
```

### Testing with Supertest

Supertest makes HTTP requests directly against a Node.js HTTP server or Express app without binding to a port — the app doesn't need to be listening. This tests the full request/response lifecycle: middleware executes, routing resolves, validation runs, and error handlers fire, exactly as in production. It's the right tool for Express route-level tests.

```typescript
import request from "supertest";
import express from "express";
import { z } from "zod";

const app = express();
app.use(express.json());

const userSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
});

app.post("/users", async (req, res) => {
  const result = userSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(400).json({ errors: result.error.errors });
  }
  const user = { id: 1, ...result.data };
  res.status(201).json(user);
});

app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});

// Tests
describe("POST /users", () => {
  it("returns 201 with created user", async () => {
    const res = await request(app)
      .post("/users")
      .send({ name: "Alice", email: "alice@example.com" })
      .expect(201);

    expect(res.body).toMatchObject({ name: "Alice", email: "alice@example.com" });
    expect(res.body.id).toBeDefined();
  });

  it("returns 400 for invalid email", async () => {
    const res = await request(app)
      .post("/users")
      .send({ name: "Alice", email: "not-an-email" })
      .expect(400);

    expect(res.body.errors).toBeDefined();
  });

  it("returns 400 when name is missing", async () => {
    await request(app)
      .post("/users")
      .send({ email: "alice@example.com" })
      .expect(400);
  });
});
```

## 16.3 NestJS

NestJS brings inversion of control to Node.js. It wraps Express (or Fastify) and adds a module system, a dependency injection container, and four lifecycle hooks that handle cross-cutting concerns before and after your route handlers run. The DI container means you declare dependencies as constructor parameters and the framework resolves and injects them — this makes unit testing clean because you can inject mocks without touching the actual module. Understanding the request lifecycle order is the core interview question: **Middleware → Guard → Interceptor (in) → Pipe → Controller → Interceptor (out) → Exception Filter**. Guards handle authorization (return true/false); Pipes transform or validate input; Interceptors can mutate the request *and* response; Filters catch exceptions and shape error responses.

### Architecture

NestJS organises code into Modules, Controllers, and Providers. `@Module` wires them: `controllers` receive HTTP requests, `providers` are DI-injectable services, and `exports` make a provider available to importing modules.

```typescript
// user.module.ts
import { Module } from "@nestjs/common";
import { UserController } from "./user.controller";
import { UserService } from "./user.service";
import { UserRepository } from "./user.repository";

@Module({
  controllers: [UserController],
  providers: [UserService, UserRepository],
  exports: [UserService], // Available to other modules
})
export class UserModule {}

// user.controller.ts
import { Controller, Get, Post, Body, Param, UseGuards } from "@nestjs/common";
import { UserService } from "./user.service";
import { JwtAuthGuard } from "../auth/jwt-auth.guard";

@Controller("users")
@UseGuards(JwtAuthGuard) // Applied to all routes
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get()
  findAll() {
    return this.userService.findAll();
  }

  @Get(":id")
  findOne(@Param("id") id: string) {
    return this.userService.findOne(+id);
  }

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.userService.create(createUserDto);
  }
}

// user.service.ts
import { Injectable } from "@nestjs/common";
import { UserRepository } from "./user.repository";

@Injectable()
export class UserService {
  constructor(private readonly userRepository: UserRepository) {}

  async findAll() {
    return this.userRepository.findAll();
  }

  async findOne(id: number) {
    const user = await this.userRepository.findById(id);
    if (!user) {
      throw new NotFoundException(`User ${id} not found`);
    }
    return user;
  }

  async create(dto: CreateUserDto) {
    return this.userRepository.create(dto);
  }
}
```

### Guards, Interceptors, Pipes, Filters

#### Guards (Authorization)

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!requiredRoles) {
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    return requiredRoles.some(role => user.roles?.includes(role));
  }
}

// Usage
@Post()
@Roles('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
create(@Body() dto: CreateUserDto) {
  // ...
}
```

#### Interceptors (Transform response, logging)

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map(data => ({
        success: true,
        data,
        timestamp: new Date().toISOString(),
      }))
    );
  }
}

// Usage
@UseInterceptors(TransformInterceptor)
@Get()
findAll() {
  return [{ id: 1, name: 'Alice' }];
}
// Response: { success: true, data: [...], timestamp: "..." }
```

#### Pipes (Validation, Transformation)

```typescript
import { PipeTransform, Injectable, BadRequestException } from '@nestjs/common';
import { z } from 'zod';

@Injectable()
export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: z.ZodSchema) {}

  transform(value: any) {
    const result = this.schema.safeParse(value);
    if (!result.success) {
      throw new BadRequestException(result.error.errors);
    }
    return result.data;
  }
}

// Usage
@Post()
create(@Body(new ZodValidationPipe(createUserSchema)) dto: CreateUserDto) {
  return this.userService.create(dto);
}
```

#### Exception Filters (Error handling)

```typescript
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
} from "@nestjs/common";

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message: exception.message,
    });
  }
}

// Global usage
app.useGlobalFilters(new HttpExceptionFilter());
```

### Microservices

NestJS's `@nestjs/microservices` package lets you run NestJS apps as microservices that communicate over a transport (TCP, Redis pub/sub, RabbitMQ, Kafka, gRPC, NATS) rather than HTTP. A microservice app listens for messages using `@MessagePattern` handlers — structurally identical to HTTP controllers but transport-agnostic. An API gateway (itself a NestJS HTTP app) calls downstream services via `ClientProxy.send()` (request-response) or `.emit()` (fire-and-forget). You can switch transports by changing a configuration option without touching business logic, and the same DI, guards, and interceptors work across HTTP and microservice contexts.

```typescript
// main.ts (microservice)
import { NestFactory } from "@nestjs/core";
import { MicroserviceOptions, Transport } from "@nestjs/microservices";
import { AppModule } from "./app.module";

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.REDIS,
      options: {
        host: "localhost",
        port: 6379,
      },
    },
  );

  await app.listen();
}

// user.controller.ts (microservice)
import { Controller } from "@nestjs/common";
import { MessagePattern } from "@nestjs/microservices";

@Controller()
export class UserController {
  @MessagePattern({ cmd: "get_user" })
  getUser(data: { id: number }) {
    return { id: data.id, name: "Alice" };
  }
}

// API Gateway (calls microservice)
import { Injectable } from "@nestjs/common";
import {
  ClientProxy,
  ClientProxyFactory,
  Transport,
} from "@nestjs/microservices";

@Injectable()
export class UserService {
  private client: ClientProxy;

  constructor() {
    this.client = ClientProxyFactory.create({
      transport: Transport.REDIS,
      options: { host: "localhost", port: 6379 },
    });
  }

  async getUser(id: number) {
    return this.client.send({ cmd: "get_user" }, { id }).toPromise();
  }
}
```

**Supported transports:** TCP, Redis, NATS, RabbitMQ, Kafka, gRPC, MQTT.

### CQRS (Command Query Responsibility Segregation)

CQRS separates write operations (Commands — change state, return nothing or the created entity) from read operations (Queries — return data, change nothing). NestJS's `@nestjs/cqrs` package provides a `CommandBus` and `QueryBus`: controllers dispatch commands/queries onto the bus, and registered handlers execute them. The pattern prevents the "service that does everything" anti-pattern and makes each handler individually testable. It's most valuable when reads and writes have very different performance, validation, or event-sourcing requirements; for a simple CRUD API it's over-engineering. It pairs naturally with the Outbox Pattern (§21) for reliable event publishing after commands.

```typescript
import { CommandHandler, ICommandHandler } from '@nestjs/cqrs';

// Command
export class CreateUserCommand {
  constructor(public readonly name: string, public readonly email: string) {}
}

// Command Handler
@CommandHandler(CreateUserCommand)
export class CreateUserHandler implements ICommandHandler<CreateUserCommand> {
  constructor(private readonly userRepository: UserRepository) {}

  async execute(command: CreateUserCommand) {
    const user = await this.userRepository.create({
      name: command.name,
      email: command.email,
    });

    // Publish event
    this.eventBus.publish(new UserCreatedEvent(user.id));

    return user;
  }
}

// Usage in controller
@Post()
async create(@Body() dto: CreateUserDto) {
  return this.commandBus.execute(new CreateUserCommand(dto.name, dto.email));
}
```

### Testing

NestJS testing follows three tiers with distinct cost/confidence tradeoffs. **Unit tests** verify a single class in isolation — all dependencies are `jest.fn()` mocks, so they run in milliseconds with no infrastructure. **Module tests** (component/integration) wire a real NestJS module with real providers but stub only the infrastructure boundaries (database, external HTTP) — they confirm the DI graph compiles and that classes in the module interact correctly. **E2E tests** boot the full HTTP application (all middleware, guards, pipes, and exception filters) and make real HTTP requests via supertest — highest confidence, slowest feedback, reserved for critical request flows.

#### Unit Tests (single class in isolation)

```typescript
import { Test, TestingModule } from "@nestjs/testing";
import { UserService } from "./user.service";
import { UserRepository } from "./user.repository";
import { NotFoundException } from "@nestjs/common";

describe("UserService", () => {
  let service: UserService;
  let repo: jest.Mocked<UserRepository>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: UserRepository,
          // Replace the real repo with a typed mock — no DB connection
          useValue: {
            findById: jest.fn(),
            findAll: jest.fn(),
            create: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get(UserService);
    repo = module.get(UserRepository);
  });

  it("throws NotFoundException when user does not exist", async () => {
    repo.findById.mockResolvedValue(null);

    await expect(service.findOne(99)).rejects.toThrow(NotFoundException);
    expect(repo.findById).toHaveBeenCalledWith(99);
  });

  it("returns user when found", async () => {
    const user = { id: 1, name: "Alice", email: "alice@example.com" };
    repo.findById.mockResolvedValue(user);

    await expect(service.findOne(1)).resolves.toEqual(user);
  });

  it("delegates create to repository", async () => {
    const dto = { name: "Alice", email: "alice@example.com" };
    repo.create.mockResolvedValue({ id: 1, ...dto });

    const result = await service.create(dto);

    expect(repo.create).toHaveBeenCalledWith(dto);
    expect(result.id).toBe(1);
  });
});
```

#### Module Tests (DI wiring, smoke tests)

Module tests wire the full NestJS module but override only the infrastructure providers — typically the TypeORM/Prisma repository or the HTTP client. The goal is to confirm the DI graph compiles (missing `@Injectable()`, wrong `provide` tokens, and circular dependencies all surface here) and that real service ↔ real repository interactions behave correctly with a fake data layer.

```typescript
import { Test, TestingModule } from "@nestjs/testing";
import { UserModule } from "./user.module";
import { UserService } from "./user.service";
import { getRepositoryToken } from "@nestjs/typeorm";
import { User } from "./user.entity";

describe("UserModule (module test)", () => {
  let service: UserService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      imports: [UserModule],   // Import the real module — real DI wiring
    })
      // Only override the TypeORM repository token — everything else is real
      .overrideProvider(getRepositoryToken(User))
      .useValue({
        findOne: jest.fn().mockResolvedValue({ id: 1, name: "Alice" }),
        save: jest.fn().mockImplementation((dto) => Promise.resolve({ id: 2, ...dto })),
        find: jest.fn().mockResolvedValue([]),
      })
      .compile();

    service = module.get(UserService);
  });

  it("DI graph resolves — all providers injectable", () => {
    // If compile() succeeds, every @Injectable() and @InjectRepository() is wired
    expect(service).toBeDefined();
  });

  it("findOne returns the user from the repository", async () => {
    const user = await service.findOne(1);
    expect(user.name).toBe("Alice");
  });
});
```

#### E2E Tests (full HTTP stack with supertest)

E2E tests start the complete NestJS application — including all globally registered middleware, guards, interceptors, pipes, and exception filters — and exercise it via HTTP. This tier catches issues the unit tests miss: a guard accidentally blocking a route, a validation pipe rejecting valid input because of a misconfigured DTO, or a missing `app.useGlobalPipes()` call that was in `main.ts` but not in the test setup.

```typescript
import { INestApplication, ValidationPipe } from "@nestjs/common";
import { Test, TestingModule } from "@nestjs/testing";
import * as request from "supertest";
import { AppModule } from "../src/app.module";

describe("UsersController (e2e)", () => {
  let app: INestApplication;

  beforeAll(async () => {
    const module: TestingModule = await Test.createTestingModule({
      imports: [AppModule],  // Full application — same as production
    }).compile();

    app = module.createNestApplication();
    // Register the same global pipes as main.ts — omitting this is a common mistake
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true }));
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  describe("POST /users", () => {
    it("creates a user and returns 201", async () => {
      const res = await request(app.getHttpServer())
        .post("/users")
        .send({ name: "Alice", email: "alice@example.com" })
        .expect(201);

      expect(res.body).toMatchObject({ name: "Alice" });
      expect(res.body.id).toBeDefined();
    });

    it("returns 400 when email is invalid — ValidationPipe fires", async () => {
      await request(app.getHttpServer())
        .post("/users")
        .send({ name: "Alice", email: "not-an-email" })
        .expect(400);
    });

    it("returns 401 when JWT guard blocks the route", async () => {
      // This verifies the guard runs — a unit test with a mocked guard wouldn't catch this
      await request(app.getHttpServer())
        .post("/users/admin-only")
        .send({ name: "Alice", email: "alice@example.com" })
        .expect(401);
    });

    it("returns 401 for a valid request with an expired token", async () => {
      await request(app.getHttpServer())
        .get("/users/me")
        .set("Authorization", "Bearer expired.jwt.token")
        .expect(401);
    });
  });
});
```

#### Database Integration (Testcontainers)

For tests that need a real database — to exercise SQL constraints, transactions, or Prisma migrations — Testcontainers starts a fresh Docker PostgreSQL container per test suite and destroys it afterwards. This eliminates "works on my machine" divergence: the test DB is always a clean, known state, and it catches constraint violations and migration issues that mocked repositories cannot.

```typescript
import { PostgreSqlContainer, StartedPostgreSqlContainer } from "@testcontainers/postgresql";
import { Test } from "@nestjs/testing";
import { TypeOrmModule } from "@nestjs/typeorm";
import { UserModule } from "./user.module";
import { UserService } from "./user.service";

let container: StartedPostgreSqlContainer;

beforeAll(async () => {
  container = await new PostgreSqlContainer().start();
}, 60_000);  // Container startup can take up to 60 s

afterAll(async () => {
  await container.stop();
});

it("persists user and retrieves by ID", async () => {
  const module = await Test.createTestingModule({
    imports: [
      TypeOrmModule.forRoot({
        type: "postgres",
        host: container.getHost(),
        port: container.getMappedPort(5432),
        username: container.getUsername(),
        password: container.getPassword(),
        database: container.getDatabase(),
        entities: [User],
        synchronize: true,  // Creates schema from entities for the test DB
      }),
      UserModule,
    ],
  }).compile();

  const service = module.get(UserService);
  const created = await service.create({ name: "Alice", email: "alice@example.com" });

  const found = await service.findOne(created.id);
  expect(found.name).toBe("Alice");

  await module.close();
});
```

## Backend Priority Summary

| Topic                                          | Priority      |
| ---------------------------------------------- | ------------- |
| **Node.js Core**                               |               |
| Event Loop phases (timers/poll/check)          | **Deep**      |
| Streams + backpressure                         | **Deep**      |
| Worker Threads                                 | **Important** |
| Node 20/22 features (test runner, .env, fetch) | **Learn**     |
| ESM vs CJS                                     | **Know**      |
| Performance profiling (--inspect, clinic.js)   | **Deep**      |
| Graceful shutdown, error handling              | **Deep**      |
| **Express**                                    |               |
| Middleware chain, error middleware             | **Refresh**   |
| Validation (Zod)                               | **Refresh**   |
| Security (Helmet, CORS, rate limiting)         | **Deep**      |
| **NestJS**                                     |               |
| Modules/Controllers/Providers/DI               | **Refresh**   |
| Guards/Interceptors/Pipes/Filters              | **Refresh**   |
| Microservices (TCP, Redis, Kafka, gRPC)        | **Important** |
| CQRS module                                    | **Learn**     |
| OpenAPI/Swagger                                | **Know**      |
| Testing (@nestjs/testing)                      | **Deep**      |

---

# 17. API Design

API design is the contract between services and the clients that consume them — get it wrong and you ship breaking changes, inconsistent error handling, and N+1 performance traps that are painful to fix in production. The landscape has fractured along use-case lines: REST remains the default for public and external APIs because of its simplicity and cacheability; GraphQL wins where client flexibility matters more than server simplicity; gRPC dominates internal service-to-service communication where throughput and strong typing are the priority; tRPC closes the loop for TypeScript full-stack apps by eliminating the client/server type boundary entirely. Knowing which to reach for, and *why*, is the interview question — not just how to implement each one. This section covers the trade-offs, the most common REST pitfalls, and the mechanisms that differentiate each protocol.

## 17.1 REST

REST (Representational State Transfer) is the architectural style that underpins the majority of web APIs. Its constraints — stateless, client-server, uniform interface, cacheable, layered — produce a style where resources are identified by URLs, representations are transferred as JSON or XML, and HTTP verbs signal intent. In practice, interviewers care about three specifics: correct HTTP verb semantics and idempotency, appropriate status codes (not everything is 200 or 500), and pagination strategy. The Richardson Maturity Model is a useful framing for assessing how "RESTful" an API actually is — most production APIs are Level 2 (resources + HTTP verbs), and Level 3 HATEOAS is rare outside hypermedia-focused systems.

### Richardson Maturity Model

The model grades API design in four levels. Most production APIs sit at L2 (resources + HTTP verbs); L3 HATEOAS is rare.

| Level  | Description          | Example                                                        |
| ------ | -------------------- | -------------------------------------------------------------- |
| **L0** | HTTP as tunnel (RPC) | POST /api { method: "getUser", params: [1] }                   |
| **L1** | Resources            | POST /users/1                                                  |
| **L2** | HTTP verbs           | GET /users/1                                                   |
| **L3** | HATEOAS              | GET /users/1 → { ..., \_links: { orders: "/users/1/orders" } } |

**Most APIs are L2.** L3 (HATEOAS) is rare in practice.

### HTTP Methods & Idempotency

Idempotency means a request can be retried without side effects. GET, PUT, and DELETE are idempotent; POST is not — submitting it twice creates two resources.

| Method  | Idempotent | Safe | Use                              |
| ------- | ---------- | ---- | -------------------------------- |
| GET     | ✅         | ✅   | Read                             |
| POST    | ❌         | ❌   | Create (non-idempotent)          |
| PUT     | ✅         | ❌   | Update (replace entire resource) |
| PATCH   | ❌\*       | ❌   | Update (partial)                 |
| DELETE  | ✅         | ❌   | Delete                           |
| OPTIONS | ✅         | ✅   | CORS preflight                   |
| HEAD    | ✅         | ✅   | Metadata only                    |

\*PATCH can be idempotent if designed carefully.

**Idempotency-Key Header:**

```http
POST /payments
Idempotency-Key: 7f8d3c21-5b1a-4f2e-8c9d-3a7b6c5d4e3f
Content-Type: application/json

{
  "amount": 100,
  "currency": "USD"
}
```

Server stores key → if duplicate request, returns same response (prevents double charges).

### Status Codes

Status codes communicate outcome semantics. Interviewers probe the difference between 400/401/403/422 — choosing the wrong one misleads API consumers and breaks client error handling.

| Code    | Meaning               | Use                                                 |
| ------- | --------------------- | --------------------------------------------------- |
| **200** | OK                    | Successful GET, PUT, PATCH                          |
| **201** | Created               | Successful POST (return resource + Location header) |
| **204** | No Content            | Successful DELETE (no body)                         |
| **400** | Bad Request           | Validation error                                    |
| **401** | Unauthorized          | Missing/invalid token                               |
| **403** | Forbidden             | Valid token, insufficient permissions               |
| **404** | Not Found             | Resource doesn't exist                              |
| **409** | Conflict              | Duplicate resource (e.g., email exists)             |
| **422** | Unprocessable Entity  | Semantic errors (valid syntax, invalid data)        |
| **429** | Too Many Requests     | Rate limit exceeded                                 |
| **500** | Internal Server Error | Server error                                        |
| **502** | Bad Gateway           | Upstream service error                              |
| **503** | Service Unavailable   | Server overloaded/maintenance                       |
| **504** | Gateway Timeout       | Upstream timeout                                    |

### Pagination

#### Offset-based

```http
GET /users?limit=20&offset=40
```

**Response:**

```json
{
  "data": [...],
  "pagination": {
    "limit": 20,
    "offset": 40,
    "total": 1000
  }
}
```

**Pros:** Simple, supports jumping to page.  
**Cons:** Inconsistent if data changes (items can shift).

#### Cursor-based (Recommended)

```http
GET /users?limit=20&cursor=eyJpZCI6MTAwfQ==
```

**Response:**

```json
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTIwfQ==",
    "hasMore": true
  }
}
```

**Pros:** Consistent (no shifting), efficient for large datasets.  
**Cons:** Can't jump to arbitrary page.

### Versioning

#### URL Versioning (Most Common)

```http
GET /v1/users
GET /v2/users
```

**Pros:** Easy to route, clear.  
**Cons:** URL changes.

#### Header Versioning

```http
GET /users
Accept: application/vnd.api.v1+json
```

**Pros:** URL stays same.  
**Cons:** Less visible, harder to cache.

#### Query Param

```http
GET /users?version=1
```

**Pros:** Simple.  
**Cons:** Clutters query params.

### API Documentation

#### OpenAPI (Swagger)

```typescript
import { ApiProperty, ApiTags } from "@nestjs/swagger";

@ApiTags("users")
@Controller("users")
export class UserController {
  @Get()
  @ApiOperation({ summary: "Get all users" })
  @ApiResponse({ status: 200, description: "Users retrieved", type: [UserDto] })
  findAll() {
    return this.userService.findAll();
  }
}

export class UserDto {
  @ApiProperty({ example: 1 })
  id: number;

  @ApiProperty({ example: "Alice" })
  name: string;

  @ApiProperty({ example: "alice@example.com" })
  email: string;
}
```

**Tools:** Swagger UI, Redoc, Scalar.

## 17.2 GraphQL

GraphQL is a query language and runtime that lets clients request exactly the fields they need — nothing more, nothing less. The server exposes a typed schema; the client writes a query against it; the server resolves each field. The upside is flexibility: no under-fetching (multiple round trips to assemble data) and no over-fetching (receiving unused fields). The downside is that the **N+1 problem** is endemic to naive implementations — resolving a list of posts and then fetching each post's author triggers one SQL query per post. DataLoader solves this by batching all the author lookups for a request into a single query, then caching the results within that request's lifetime.

### Schema

The schema is the contract between client and server. Every field is explicitly typed, nullability is declared with `!` (non-nullable), and the schema itself documents the available queries and mutations.

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
}

type Query {
  user(id: ID!): User
  users: [User!]!
  post(id: ID!): Post
}

type Mutation {
  createUser(name: String!, email: String!): User!
  updateUser(id: ID!, name: String): User!
}
```

### Resolvers (Node.js)

Each schema field maps to a resolver function. Resolvers are called lazily — the runtime only invokes them for fields the client's query actually selected.

```typescript
const resolvers = {
  Query: {
    user: (parent, { id }, context) => {
      return context.db.users.findById(id);
    },
    users: (parent, args, context) => {
      return context.db.users.findAll();
    },
  },

  Mutation: {
    createUser: (parent, { name, email }, context) => {
      return context.db.users.create({ name, email });
    },
  },

  User: {
    posts: (user, args, context) => {
      return context.db.posts.findByAuthorId(user.id);
    },
  },
};
```

### N+1 Problem (Solved with DataLoader)

Without DataLoader, resolving N users and fetching each user's posts fires N+1 queries. DataLoader batches all `.load(id)` calls made during a single tick into one query, then caches results for the lifetime of the request.

```typescript
import DataLoader from "dataloader";

// Without DataLoader: N+1 queries
// users.forEach(user => db.posts.findByAuthorId(user.id)); // N queries

// With DataLoader: 1 query
const postLoader = new DataLoader(async (authorIds) => {
  const posts = await db.posts.findByAuthorIds(authorIds); // 1 query

  // Group by authorId
  const postsByAuthor = {};
  posts.forEach((post) => {
    if (!postsByAuthor[post.authorId]) postsByAuthor[post.authorId] = [];
    postsByAuthor[post.authorId].push(post);
  });

  return authorIds.map((id) => postsByAuthor[id] || []);
});

const resolvers = {
  User: {
    posts: (user, args, context) => {
      return context.loaders.postLoader.load(user.id);
    },
  },
};
```

### GraphQL vs REST

The choice depends on flexibility requirements, caching needs, and the number of clients. GraphQL shines for mobile apps and complex data requirements; REST is simpler for public APIs.

|                        | GraphQL                                | REST                                   |
| ---------------------- | -------------------------------------- | -------------------------------------- |
| **Data fetching**      | Exact fields (no over/under-fetching)  | Fixed endpoints (over-fetching common) |
| **Multiple resources** | Single request                         | Multiple requests or custom endpoints  |
| **Versioning**         | Evolve schema (deprecate fields)       | URL or header versioning               |
| **Caching**            | Complex (query-based)                  | Simple (HTTP caching)                  |
| **File uploads**       | Complex                                | Simple (multipart/form-data)           |
| **Learning curve**     | Steeper                                | Simpler                                |
| **Best for**           | Mobile apps, complex data requirements | Simple CRUD, public APIs               |

### Security

#### Query Depth Limiting

```typescript
import depthLimit from "graphql-depth-limit";

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [depthLimit(5)], // Max 5 levels deep
});
```

#### Query Cost Analysis

```typescript
import { createComplexityLimitRule } from "graphql-validation-complexity";

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    createComplexityLimitRule(1000, {
      scalarCost: 1,
      objectCost: 10,
      listFactor: 10,
    }),
  ],
});
```

## 17.3 gRPC

gRPC is Google's open-source RPC framework built on Protocol Buffers and HTTP/2. The `.proto` file is the source of truth — it defines services and message types, and the gRPC toolchain generates typed client and server stubs in any supported language. Because Protocol Buffers serialize to binary (not text), messages are significantly smaller and faster to parse than equivalent JSON. HTTP/2 multiplexing means many concurrent RPC calls share one TCP connection without head-of-line blocking. The result: gRPC is the standard for internal service-to-service communication at scale; REST is preferred for public APIs where human readability, browser support, and universal client tooling matter more.

### Proto Definition

The `.proto` file is the source of truth — it defines services, RPCs, and message types. Client and server stubs are code-generated from it in any supported language.

```protobuf
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);
  rpc CreateUser (CreateUserRequest) returns (User);
}

message GetUserRequest {
  int32 id = 1;
}

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
}

message ListUsersRequest {
  int32 limit = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}
```

### gRPC vs REST

gRPC's binary encoding and HTTP/2 multiplexing make it significantly faster than JSON/REST, but it requires a proxy for browser clients and is harder to inspect with standard HTTP tools.

|                 | gRPC                              | REST                         |
| --------------- | --------------------------------- | ---------------------------- |
| Protocol        | Protobuf (binary)                 | JSON (text)                  |
| Speed           | Faster (smaller, binary)          | Slower (larger, text)        |
| Streaming       | Bidirectional                     | Server-Sent Events (one-way) |
| Browser support | Requires proxy (grpc-web)         | Native                       |
| Human-readable  | No                                | Yes                          |
| Best for        | Service-to-service, microservices | Public APIs, browser clients |

## 17.4 tRPC

tRPC is a TypeScript-first RPC framework for monorepos where the frontend and backend share a codebase. Rather than generating code from a schema file, it infers types directly from your router definitions at build time — the client gets full autocompletion and type safety with zero codegen. A procedure defined on the server is callable on the client as if it were a local function, with input/output types fully inferred. The trade-off is that both sides must be TypeScript, making tRPC unsuitable for public APIs or polyglot stacks where consumers are in other languages.

### tRPC Server

Procedures are defined as queries (reads) or mutations (writes). The exported `AppRouter` type is all the client needs for full end-to-end type safety — no code generation step required.

```typescript
import { initTRPC } from "@trpc/server";
import { z } from "zod";

const t = initTRPC.create();

const appRouter = t.router({
  getUser: t.procedure
    .input(z.object({ id: z.number() }))
    .query(async ({ input }) => {
      return db.users.findById(input.id);
    }),

  createUser: t.procedure
    .input(z.object({ name: z.string(), email: z.string().email() }))
    .mutation(async ({ input }) => {
      return db.users.create(input);
    }),
});

export type AppRouter = typeof appRouter;
```

### tRPC Client (Next.js)

The client imports only the router's *type* (not the implementation) and infers all procedure types at compile time. Calling a procedure feels like a local function call; the underlying HTTP request and batching are invisible.

```typescript
import { createTRPCProxyClient, httpBatchLink } from "@trpc/client";
import type { AppRouter } from "./server";

const trpc = createTRPCProxyClient<AppRouter>({
  links: [
    httpBatchLink({
      url: "http://localhost:3000/api/trpc",
    }),
  ],
});

// Fully typed!
const user = await trpc.getUser.query({ id: 1 });
//    ^? { id: number; name: string; email: string }

const newUser = await trpc.createUser.mutate({
  name: "Alice",
  email: "alice@example.com",
});
```

**Why tRPC:**

- End-to-end type safety
- No code generation
- Works with monorepos
- Best for full-stack TypeScript apps

## API Design Priority Summary

| Topic                                                                | Priority      |
| -------------------------------------------------------------------- | ------------- |
| **REST**                                                             |               |
| Richardson Maturity (L0-L3)                                          | **Know**      |
| Idempotency (GET, PUT, DELETE)                                       | **Critical**  |
| Status codes (200, 201, 204, 400, 401, 403, 404, 409, 422, 429, 500) | **Critical**  |
| Pagination (offset vs cursor)                                        | **Deep**      |
| Versioning (URL, header, query)                                      | **Important** |
| OpenAPI/Swagger                                                      | **Learn**     |
| **GraphQL**                                                          |               |
| Schema, resolvers                                                    | **Learn**     |
| N+1 problem (DataLoader)                                             | **Important** |
| Query depth/cost limiting                                            | **Learn**     |
| GraphQL vs REST trade-offs                                           | **Deep**      |
| **gRPC**                                                             |               |
| Protobuf, service definition                                         | **Know**      |
| gRPC vs REST                                                         | **Know**      |
| **tRPC**                                                             |               |
| End-to-end type safety                                               | **Learn**     |
| **OData**                                                            |               |
| $filter/$select/$expand query language                               | **Know**      |
| **BFF (Backend for Frontend)**                                       |               |
| BFF vs API Gateway distinction                                       | **Deep**      |
| Aggregation + parallel downstream calls                              | **Important** |
| GraphQL as BFF                                                       | **Important** |
| **End-to-End Type Safety**                                           |               |
| tRPC vs OpenAPI codegen vs GraphQL codegen                           | **Deep**      |
| openapi-typescript / orval                                           | **Important** |
| graphql-codegen                                                      | **Learn**     |

---

## 17.5 OData

OData (Open Data Protocol) is a REST-based protocol standardised by OASIS that extends REST with a URL-based query language. Instead of building custom endpoints for every filtering need, OData exposes a uniform query interface via URL query parameters. It's common in Microsoft/Azure ecosystems — Azure DevOps API, SharePoint, Dynamics 365, and many enterprise BI tools consume OData.

**Why it matters:** If you work in enterprise environments or with Azure, you'll encounter OData APIs. Understanding the query language saves time when integrating with these systems.

**Core query options:**
- `$filter` — filter records: `Price gt 20 and Category eq 'Electronics'`
- `$select` — specify fields to return (like SQL SELECT)
- `$expand` — include related entities (like SQL JOIN)
- `$orderby` — sort: `Price desc`
- `$top` / `$skip` — pagination
- `$count` — return total record count

```http
# Get the 10 cheapest in-stock electronics, sorted by price ascending
GET /odata/Products
  ?$filter=Category eq 'Electronics' and InStock eq true
  &$select=Name,Price,Category
  &$orderby=Price asc
  &$top=10
  &$skip=0
  &$count=true
```

**When to use OData:** Enterprise integrations, BI tooling, scenarios where clients need ad-hoc querying of large datasets without endpoint proliferation. OData is NOT recommended for consumer APIs — it's complex for clients to consume and exposes too much of your data model.

---

## 17.6 Backend for Frontend (BFF)

The **Backend for Frontend** pattern — coined by Sam Newman — places a dedicated server-side aggregation layer between a specific frontend client and the downstream services it consumes. Instead of the frontend calling five microservices and assembling the data itself, a BFF calls all five, aggregates, transforms, and shapes the response for exactly what that UI needs. Network chattiness moves to the server-to-server layer where latency is low and retries are cheap. This section covers when BFF makes sense, what it looks like in a NestJS stack, and how to draw the boundary between a BFF and an API gateway — a distinction interviewers frequently probe.

### BFF vs API Gateway

They are not alternatives — the canonical setup has both. The API Gateway handles the perimeter; the BFF handles client-specific aggregation behind it.

| | API Gateway | BFF |
| --- | --- | --- |
| **Purpose** | Cross-cutting concerns (auth, rate limiting, routing, SSL termination) | Client-specific data aggregation & transformation |
| **Scope** | All clients | One client type (web, mobile, TV app) |
| **Data logic** | None — routes and proxies | Aggregates, filters, reshapes |
| **Owned by** | Platform / infra team | The frontend / product team |
| **Examples** | AWS API Gateway, Kong, nginx | A Node.js / NestJS service |

**The canonical setup is both:** the API Gateway handles the perimeter; the BFF handles client-specific composition. They are not alternatives — the gateway routes traffic to the BFF.

### When to use a BFF

Use a BFF when:
- A single screen requires aggregating data from 3+ downstream services
- Web and mobile clients need significantly different data shapes from the same resources
- The frontend team wants ownership of their API contract without depending on backend service teams
- You need to shield the frontend from internal service decomposition

Avoid a BFF when you have a single client, a monolith, or simple CRUD — the extra service adds latency and operational overhead with no benefit.

### Implementation (NestJS)

The BFF service fires downstream calls in parallel with `Promise.all`, shapes a composite DTO from the results, and owns authentication at its own boundary.

```typescript
// bff/src/product-page/product-page.service.ts
@Injectable()
export class ProductPageService {
  constructor(
    private readonly catalogClient: CatalogClient,
    private readonly inventoryClient: InventoryClient,
    private readonly reviewsClient: ReviewsClient,
  ) {}

  async getProductPage(productId: string): Promise<ProductPageDto> {
    // All three downstream calls fire in parallel
    const [product, inventory, reviews] = await Promise.all([
      this.catalogClient.getProduct(productId),
      this.inventoryClient.getStock(productId),
      this.reviewsClient.getTopReviews(productId, { limit: 5 }),
    ]);

    // Shape the response for exactly what the UI needs — nothing more
    return {
      id: product.id,
      name: product.name,
      price: product.price,
      inStock: inventory.quantity > 0,
      rating: reviews.averageRating,
      topReviews: reviews.items.map(r => ({ author: r.authorName, body: r.body })),
    };
  }
}
```

```typescript
// bff/src/product-page/product-page.controller.ts
@Controller('product-page')
@UseGuards(JwtAuthGuard)   // Auth lives at the BFF boundary
export class ProductPageController {
  constructor(private readonly service: ProductPageService) {}

  @Get(':id')
  getProductPage(@Param('id') id: string) {
    return this.service.getProductPage(id);
  }
}
```

The BFF owns auth, session handling, and response shaping. Downstream services receive service-to-service authenticated requests (service accounts, mTLS) — they never see user JWTs directly.

### GraphQL as BFF

GraphQL is a natural BFF implementation: the schema is the frontend-specific contract, resolvers aggregate downstream services, and clients query exactly the shape they need without over-fetching.

```typescript
@Resolver(() => ProductPage)
export class ProductPageResolver {
  @Query(() => ProductPage)
  async productPage(@Args('id') id: string): Promise<ProductPage> {
    const [product, inventory, reviews] = await Promise.all([
      this.catalogClient.getProduct(id),
      this.inventoryClient.getStock(id),
      this.reviewsClient.getTopReviews(id, { limit: 5 }),
    ]);
    return { ...product, inStock: inventory.quantity > 0, topReviews: reviews.items };
  }
}
```

Many teams use Apollo Server or `NestJS GraphQLModule` as their BFF layer. The GraphQL schema becomes the versioned contract between the frontend team and the BFF.

**Interview framing:** "How does a BFF relate to an API Gateway?" — The gateway handles the perimeter (auth enforcement, rate limiting, TLS, routing); the BFF handles client-specific aggregation and shaping. They sit at different layers and are owned by different teams. A common follow-up: "What's the cost?" — you now maintain an extra service, a contract, and its deployment pipeline.

---

## 17.7 End-to-End Type Safety

Without a deliberate type-sharing strategy, frontend and backend types drift: a field renamed from `userName` to `user_name` on the server breaks the UI at runtime, not at compile time. Three strategies are in active use to eliminate this class of bug — each trades off ergonomics, tooling complexity, and polyrepo compatibility differently. Choosing the right one is an architectural decision senior engineers are regularly asked to explain.

### The problem: manual type maintenance

Without a type-sharing strategy, frontend and backend types drift silently — a renamed field breaks the UI at runtime, not at compile time.

```typescript
// Backend DTO — someone renames this field
export class UserDto {
  id: string;
  user_name: string;   // ← renamed from userName
  email: string;
}

// Frontend type — not updated
interface User {
  id: string;
  userName: string;    // ← still the old name; runtime bug, no compile error
  email: string;
}
```

### Strategy 1 — tRPC (TypeScript monorepo, no code generation)

Already covered in §17.4. The server router *is* the schema; the client infers types from it via a shared package. No build step required — types are always in sync by construction.

**Best fit:** Full-stack TypeScript monorepo (Next.js + tRPC is the canonical example).
**Not a fit:** Polyrepo, non-TS clients, or public APIs consumed by external parties.

### Strategy 2 — OpenAPI codegen (REST APIs)

Define an OpenAPI 3.x spec as the **single source of truth**, then generate TypeScript types and HTTP client code from it. Changing the spec cascades to generated types, which break dependent code at compile time.

**Tools:**
- **`openapi-typescript`** — generates TypeScript types only (no runtime client)
- **`orval`** — generates types + TanStack Query / SWR / Axios hooks (full client)

```yaml
# openapi.yaml
paths:
  /users/{id}:
    get:
      operationId: getUserById
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: string, format: uuid }
      responses:
        '200':
          content:
            application/json:
              schema: { $ref: '#/components/schemas/User' }
components:
  schemas:
    User:
      type: object
      required: [id, email, userName]
      properties:
        id:       { type: string, format: uuid }
        email:    { type: string, format: email }
        userName: { type: string }
```

```bash
# Generate types only
npx openapi-typescript ./openapi.yaml --output ./src/types/api.d.ts

# Generate types + React Query hooks with orval
npx orval --config orval.config.ts
```

```typescript
// Generated type — never edit manually
import type { components } from './types/api.d.ts';
type User = components['schemas']['User'];
//   ^ { id: string; email: string; userName: string }

// orval also generates typed React Query hooks:
const { data: user } = useGetUserById({ id: userId });
//        user is typed as User | undefined — no manual interface needed
```

**Best fit:** REST APIs, polyrepo teams, public APIs with external consumers (generate Python/Go clients from the same spec).

### Strategy 3 — GraphQL codegen

Define a GraphQL schema on the backend; write typed queries/mutations in the frontend. **`graphql-codegen`** reads both and generates TypeScript types for every operation, typed to the *exact fields selected* — not the full server-side type.

```graphql
# frontend: src/queries/getUser.graphql
query GetUser($id: ID!) {
  user(id: $id) {
    id
    email
    userName
  }
}
```

```bash
npx graphql-codegen --config codegen.ts
```

```typescript
// Generated — typed to the exact fields selected in the query
import { useGetUserQuery } from './generated/graphql';

function UserProfile({ id }: { id: string }) {
  const { data } = useGetUserQuery({ variables: { id } });
  // data.user is: { id: string; email: string; userName: string }
  // — NOT the full server User type, only what this query fetches
}
```

This is the key advantage over OpenAPI: if you add a field to the query that doesn't exist in the schema, you get a compile error immediately.

**Best fit:** GraphQL APIs, teams already using Apollo Client or urql.

### Strategy comparison

Choose tRPC for TypeScript monorepos, OpenAPI codegen for polyrepo REST APIs or public APIs, and GraphQL codegen for existing GraphQL systems.

| | tRPC | OpenAPI codegen | GraphQL codegen |
| --- | --- | --- | --- |
| **Schema required?** | No — router is the schema | Yes (OpenAPI YAML) | Yes (GraphQL schema) |
| **Code gen step?** | No | Yes | Yes |
| **Non-TS clients?** | No | Yes | Yes |
| **Polyrepo?** | Hard | Yes | Yes |
| **Per-query types?** | No | No | Yes |
| **Runtime validation** | Zod on server | Bring your own | Bring your own |

**Interview framing:** "How do you keep frontend and backend types in sync?" — Walk through the three options and their trade-offs. The anti-pattern to name explicitly: "we write them in both places and keep them in sync manually" — that's the status quo that the question is designed to uncover. Close with your recommendation: tRPC for a greenfield TS monorepo; OpenAPI codegen for a REST API or polyrepo; GraphQL codegen if already on GraphQL.

---


# 18. Authentication and Authorization

Authentication answers "who are you?" and authorization answers "what are you allowed to do?" — they are consistently the highest-priority security topics in senior interviews because misimplementing either has direct, exploitable consequences. JWT-based auth is the dominant stateless pattern: the server signs a token at login; the client presents it on every subsequent request; the server verifies the signature without touching a database. OAuth 2.0 is a delegation framework — it's how "Login with Google" works, and understanding the Authorization Code + PKCE flow versus Client Credentials is mandatory for any backend role. RBAC and ABAC describe how permissions are structured once identity is established. This section covers each mechanism, its failure modes, and the attack surfaces interviewers probe.

## 18.1 JWT (JSON Web Tokens)

A JSON Web Token is a self-contained, signed string that encodes claims — user ID, roles, expiry time — in a base64url-encoded JSON payload. The server signs it on login; the client presents it in subsequent requests; the server verifies the signature using a secret (HS256) or a public key (RS256) without touching a database. The critical security property — and the one interviewers probe — is that JWTs are *stateless*: once issued, they cannot be revoked before expiry without additional infrastructure such as a token denylist or very short TTLs paired with refresh-token rotation. Refresh-token rotation is the correct pattern: the access token is short-lived (15 min), the refresh token is longer-lived (7 days) and stored in an httpOnly cookie, and exchanging a refresh token for new tokens immediately invalidates the old refresh token — so a stolen refresh token can only be used once before detection.

### Structure

A JWT has three base64url-encoded, dot-separated parts: header (algorithm), payload (claims), and signature. Only the signature is secret — the header and payload are readable by anyone who intercepts the token.

```
header.payload.signature
```

```json
// Header
{
  "alg": "HS256",
  "typ": "JWT"
}

// Payload
{
  "sub": "1234567890",
  "name": "Alice",
  "iat": 1516239022,
  "exp": 1516242622
}

// Signature
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

### Access + Refresh Tokens

The secure pattern: short-lived access token (15 min) in memory, long-lived refresh token (7 days) in an httpOnly cookie. Exchanging a refresh token immediately invalidates the old one — a stolen refresh token can only be used once before detection.

```typescript
// Generate tokens
function generateTokens(userId: number) {
  const accessToken = jwt.sign(
    { sub: userId, type: "access" },
    process.env.JWT_SECRET,
    { expiresIn: "15m" },
  );

  const refreshToken = jwt.sign(
    { sub: userId, type: "refresh" },
    process.env.REFRESH_SECRET,
    { expiresIn: "7d" },
  );

  return { accessToken, refreshToken };
}

// Login endpoint
app.post("/auth/login", async (req, res) => {
  const { email, password } = req.body;
  const user = await authenticateUser(email, password);

  if (!user) {
    return res.status(401).json({ error: "Invalid credentials" });
  }

  const { accessToken, refreshToken } = generateTokens(user.id);

  // Store refresh token in httpOnly cookie
  res.cookie("refreshToken", refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "strict",
    maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
  });

  res.json({ accessToken });
});

// Refresh endpoint
app.post("/auth/refresh", async (req, res) => {
  const refreshToken = req.cookies.refreshToken;

  if (!refreshToken) {
    return res.status(401).json({ error: "No refresh token" });
  }

  try {
    const payload = jwt.verify(refreshToken, process.env.REFRESH_SECRET);

    if (payload.type !== "refresh") {
      return res.status(401).json({ error: "Invalid token type" });
    }

    const { accessToken, refreshToken: newRefreshToken } = generateTokens(
      payload.sub,
    );

    // Rotate refresh token
    res.cookie("refreshToken", newRefreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === "production",
      sameSite: "strict",
      maxAge: 7 * 24 * 60 * 60 * 1000,
    });

    res.json({ accessToken });
  } catch (err) {
    return res.status(401).json({ error: "Invalid token" });
  }
});

// Protected route
app.get("/api/profile", authenticateJWT, (req, res) => {
  res.json({ user: req.user });
});

// Middleware
function authenticateJWT(req, res, next) {
  const authHeader = req.headers.authorization;
  const token = authHeader?.split(" ")[1]; // "Bearer <token>"

  if (!token) {
    return res.status(401).json({ error: "No token provided" });
  }

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);

    if (payload.type !== "access") {
      return res.status(401).json({ error: "Invalid token type" });
    }

    req.user = payload;
    next();
  } catch (err) {
    return res.status(401).json({ error: "Invalid token" });
  }
}
```

### Token Storage

The trade-off is XSS risk (localStorage) vs CSRF risk (httpOnly cookies). Best practice: access token in memory (short-lived, lost on reload), refresh token in an httpOnly + sameSite cookie.

| Storage                  | XSS Vulnerable | CSRF Vulnerable                 | Use Case                                |
| ------------------------ | -------------- | ------------------------------- | --------------------------------------- |
| **localStorage**         | ✅ Yes         | ❌ No                           | Not recommended                         |
| **Cookie (httpOnly)**    | ❌ No          | ✅ Yes (mitigate with sameSite) | Recommended for refresh tokens          |
| **Memory (React state)** | ❌ No          | ❌ No                           | Best for access tokens (lost on reload) |

**Best practice:**

- Access token: Memory (short-lived, 15 min)
- Refresh token: httpOnly cookie (long-lived, 7 days)

## 18.2 OAuth 2.0 / OpenID Connect

OAuth 2.0 is a delegation framework — it lets a user authorize a third-party application to access their resources without sharing their credentials. OpenID Connect (OIDC) builds on OAuth to add *identity*: the ID token is a JWT that proves who the user is, not just what they're authorized to access. The **Authorization Code + PKCE** flow is the correct choice for any app with a browser: the auth code returned to the redirect URI is short-lived and single-use, and PKCE (Proof Key for Code Exchange) prevents an intercepted auth code from being exchanged without the original code verifier. **Client Credentials** is for machine-to-machine communication where there is no user — a backend service authenticating to another backend service.

### Authorization Code Flow + PKCE

PKCE prevents authorization code interception on mobile apps and SPAs. The client generates a code verifier, sends a hash (the challenge) with the authorization request, and proves possession by submitting the original verifier when exchanging the code for tokens.

```
1. Client generates code_verifier (random string)
2. Client computes code_challenge = base64url(sha256(code_verifier))
3. Client redirects to /authorize?
     response_type=code
     &client_id=...
     &redirect_uri=...
     &code_challenge=...
     &code_challenge_method=S256
4. User logs in, grants permission
5. Server redirects to redirect_uri?code=...
6. Client exchanges code for tokens:
   POST /token
     grant_type=authorization_code
     &code=...
     &redirect_uri=...
     &code_verifier=...
7. Server verifies code_challenge matches sha256(code_verifier)
8. Server returns access_token, id_token (OIDC), refresh_token
```

**PKCE prevents:** Authorization code interception attacks (especially on mobile/SPA).

### OpenID Connect (OIDC) = OAuth 2.0 + Identity

OIDC adds authentication (who the user is) on top of OAuth 2.0's authorization (what they can access). The ID token is a signed JWT with standard identity claims: `sub`, `email`, `name`, `picture`.

- **OAuth 2.0:** Authorization (access to resources)
- **OIDC:** Authentication (who the user is)

**OIDC adds:**

- `id_token` (JWT with user info)
- `/userinfo` endpoint
- Standard claims (sub, name, email, picture)

## 18.3 RBAC vs ABAC

Role-Based Access Control assigns permissions to roles and roles to users — simple, predictable, and fast to evaluate because the role lookup is done at login or token issuance. Attribute-Based Access Control evaluates a policy against attributes of the user (department, clearance level), the resource (classification, owner), and the environment (time of day, IP address) at request time. RBAC is the right default for most applications; ABAC is warranted when role explosion becomes real (hundreds of roles to cover every combination of conditions) or when decisions must depend on runtime context that can't be encoded into a static role. The practical middle ground is often **ReBAC** (Relationship-Based, as in Google Zanzibar / SpiceDB) — permissions based on the relationship between user and resource.

### RBAC (Role-Based Access Control)

RBAC assigns permissions to roles and roles to users. Permission checks look up the user's role at request time and check whether the required action is included.

```typescript
enum Role {
  ADMIN = "admin",
  EDITOR = "editor",
  VIEWER = "viewer",
}

const permissions = {
  [Role.ADMIN]: ["read", "write", "delete", "manage_users"],
  [Role.EDITOR]: ["read", "write"],
  [Role.VIEWER]: ["read"],
};

function hasPermission(user: User, action: string): boolean {
  const userPermissions = permissions[user.role] || [];
  return userPermissions.includes(action);
}

// Usage
if (hasPermission(user, "delete")) {
  deletePost(postId);
}
```

**Pros:** Simple, easy to manage.  
**Cons:** Coarse-grained, doesn't handle complex rules.

### ABAC (Attribute-Based Access Control)

ABAC evaluates a policy against attributes of the user, the resource, and the environment at request time — enabling fine-grained rules that would require hundreds of RBAC roles to encode statically.

```typescript
type Policy = {
  action: string;
  resource: string;
  condition: (user: User, resource: any) => boolean;
};

const policies: Policy[] = [
  {
    action: "delete",
    resource: "post",
    condition: (user, post) =>
      user.id === post.authorId || user.role === "admin",
  },
  {
    action: "edit",
    resource: "post",
    condition: (user, post) => user.id === post.authorId,
  },
];

function can(user: User, action: string, resource: string, data: any): boolean {
  const policy = policies.find(
    (p) => p.action === action && p.resource === resource,
  );
  return policy ? policy.condition(user, data) : false;
}

// Usage
if (can(user, "delete", "post", post)) {
  deletePost(post.id);
}
```

**Pros:** Fine-grained, flexible.  
**Cons:** Complex to manage.

**Libraries:** Casbin, CASL.

## 18.4 Passkeys / WebAuthn

WebAuthn (Web Authentication) is the W3C standard for passwordless, phishing-resistant authentication. The user's authenticator — a device TPM chip, Touch ID, Face ID, or a hardware security key — generates an asymmetric key pair per origin at registration. The private key never leaves the device; the server stores only the public key. Authentication works by signing a server-provided challenge: the signature proves possession of the private key without transmitting it, and the origin binding prevents the credential from being used on a phishing site. **Passkeys** are a user-friendly packaging of WebAuthn: device-synced credentials backed by iCloud Keychain, Google Password Manager, or a password manager, making the same phishing-resistant credential available across a user's devices.

### WebAuthn Registration

Registration creates an asymmetric key pair on the user's device. The private key never leaves the device; the server stores only the credential ID and the corresponding public key.

```typescript
// Server generates challenge
const challenge = crypto.randomBytes(32);
res.json({ challenge: base64url(challenge) });

// Client creates credential
const credential = await navigator.credentials.create({
  publicKey: {
    challenge: base64urlDecode(challenge),
    rp: { name: "Example Corp" },
    user: {
      id: Uint8Array.from(userId, (c) => c.charCodeAt(0)),
      name: "alice@example.com",
      displayName: "Alice",
    },
    pubKeyCredParams: [{ alg: -7, type: "public-key" }],
    authenticatorSelection: {
      authenticatorAttachment: "platform", // Built-in (Face ID, Touch ID)
      userVerification: "required",
    },
  },
});

// Client sends credential to server
// Server stores publicKey, credentialId
```

### WebAuthn Authentication

Authentication signs a server-issued challenge using the device's private key. The server verifies the signature with the stored public key — no password is transmitted or stored.

```typescript
// Server generates challenge
const challenge = crypto.randomBytes(32);
res.json({ challenge: base64url(challenge) });

// Client gets credential
const credential = await navigator.credentials.get({
  publicKey: {
    challenge: base64urlDecode(challenge),
    allowCredentials: [{ id: credentialId, type: "public-key" }],
  },
});

// Server verifies signature with stored publicKey
```

**Benefits:** Phishing-resistant, no passwords, biometric auth.

## 18.5 Providers Comparison

| Provider          | Pros                               | Cons                    | Use Case                         |
| ----------------- | ---------------------------------- | ----------------------- | -------------------------------- |
| **Auth.js**       | Free, self-hosted, flexible        | Setup complexity        | Full control, OSS projects       |
| **Clerk**         | Beautiful UI, easy setup, webhooks | $$$                     | Fast prototyping, startups       |
| **Auth0**         | Enterprise features, compliance    | $$$                     | Large orgs, complex requirements |
| **AWS Cognito**   | Cheap, integrates with AWS         | Complex API, limited UI | AWS-heavy stacks                 |
| **Supabase Auth** | Free tier, built-in with Supabase  | Less flexible           | Supabase projects                |

## Authentication Priority Summary

| Topic                                      | Priority     |
| ------------------------------------------ | ------------ |
| JWT (access + refresh, rotation, httpOnly) | **Deep**     |
| OAuth 2.0 / OIDC (Auth Code + PKCE)        | **Critical** |
| RBAC vs ABAC                               | **Know**     |
| Passkeys / WebAuthn                        | **Know**     |
| Providers (Auth0, Clerk, Auth.js, Cognito) | **Know**     |

---


# 19. Databases

Database choice and optimization are among the most heavily tested backend topics at the senior level — interviewers assume you can write queries, so they go deeper: *why* does this index help, what does `EXPLAIN ANALYZE` actually tell you, when does eventual consistency become a problem, and which paradigm fits this access pattern? This section is organized in three groups: **SQL** (fundamentals first, then PostgreSQL, MySQL, and ORMs), **NoSQL** (MongoDB, Redis, DynamoDB, Cassandra, Elasticsearch), and **cross-cutting patterns** (multi-tenancy and caching). The right choice always starts with access patterns — relational for transactional consistency and complex queries, document for hierarchical data and schema flexibility, key-value for sub-millisecond reads at scale.

### SQL

## 19.1 SQL Fundamentals

SQL (Structured Query Language) is the universal language for relational databases. A solid grasp of joins, aggregations, transactions, and query planning is expected of any senior engineer, regardless of specialisation.

### JOINs

JOINs combine rows from multiple tables on a matching condition. INNER returns only matching rows; LEFT returns all rows from the left table plus NULLs for non-matching right rows.

```sql
-- Sample tables
-- users: id, name, email
-- orders: id, user_id, total, status
-- order_items: id, order_id, product_id, quantity

-- INNER JOIN: only rows with matches in BOTH tables
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;
-- ← users with NO orders are excluded

-- LEFT JOIN: ALL rows from left table + matching rows from right
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;
-- ← users with no orders appear with order_count = 0

-- RIGHT JOIN: ALL rows from right table + matching rows from left
-- (less common — can usually be rewritten as a LEFT JOIN)

-- FULL OUTER JOIN: ALL rows from BOTH tables
SELECT u.name, o.id as order_id
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;
-- ← includes users with no orders AND orders with no matching user

-- Self-join: join a table to itself
SELECT e.name as employee, m.name as manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
-- ← employees without managers (CEO) appear with manager = NULL
```

### ACID Properties

ACID guarantees make relational databases reliable for financial and mission-critical data:

| Property    | Meaning                                                                 |
|-------------|-------------------------------------------------------------------------|
| **A**tomicity | A transaction is all-or-nothing. If any step fails, all are rolled back. |
| **C**onsistency | A transaction moves the database from one valid state to another. Constraints and rules are enforced. |
| **I**solation | Concurrent transactions behave as if they ran sequentially. One transaction can't see another's intermediate state. |
| **D**urability | Once committed, data survives crashes. Written to disk/WAL before acknowledging success. |

```sql
-- Classic bank transfer: ACID in practice
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- If this succeeds but the next fails, ROLLBACK undoes both

UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;  -- Both updates are atomically committed
-- OR: ROLLBACK;  -- Neither update persists
```

### Transaction Isolation Levels

Isolation prevents concurrent transactions from interfering. Higher isolation = fewer anomalies but lower throughput:

| Problem              | Description                                                         |
|----------------------|---------------------------------------------------------------------|
| **Dirty read**       | Transaction reads uncommitted changes from another transaction      |
| **Non-repeatable read** | Same row read twice in a transaction returns different values    |
| **Phantom read**     | A range query returns different rows when repeated (new rows added) |

PostgreSQL's default (`READ COMMITTED`) prevents dirty reads. Most financial systems use `SERIALIZABLE` for correctness.

## 19.2 MySQL

MySQL is the most widely deployed open-source relational database — the M in the LAMP/LEMP stack, and still the default at many organisations running legacy Node.js or PHP backends. InnoDB, MySQL's default storage engine since version 5.5, is fully ACID-compliant with row-level locking. For greenfield projects PostgreSQL is usually the better default (richer feature set, more correct SQL behaviour out of the box), but MySQL is pervasive in existing systems and runs at massive scale (Meta, Twitter, Airbnb all ran MySQL).

### Storage engines

InnoDB is the only viable production engine — ACID-compliant with row-level locking. Its clustered-index architecture (rows stored inside the PK B-tree) means secondary index lookups require two traversals.

| Engine | ACID | Row locking | When to use |
|---|---|---|---|
| **InnoDB** | Yes | Yes | All production workloads — the only real option |
| MyISAM | No | Table-level | Legacy; read-heavy with no transaction requirements |
| MEMORY | No | Table-level | Temporary, session-scoped data only |

**InnoDB internals — the clustered index:** Unlike PostgreSQL (heap storage + separate indexes), InnoDB stores rows inside the primary-key B-tree. This means a PK lookup is one B-tree traversal that returns the full row; a secondary index lookup requires two traversals (secondary → PK → data). Consequence: choose an auto-increment integer PK to keep inserts sequential and avoid page fragmentation.

### Replication

MySQL streams changes via the **binary log (binlog)** from a primary to one or more replicas:

```sql
-- my.cnf on primary
server-id             = 1
log_bin               = mysql-bin
binlog_format         = ROW   -- row-based: deterministic, recommended over statement-based
gtid_mode             = ON    -- Global Transaction IDs: simpler failover, no log offsets
enforce_gtid_consistency = ON

-- On each replica after connecting it to the primary:
START REPLICA;
SHOW REPLICA STATUS\G   -- check Seconds_Behind_Source and any replication errors
```

**Statement vs row-based binlog:**
- **Statement-based:** logs the SQL — compact, but non-deterministic functions (`NOW()`, `UUID()`) can produce different values on replicas.
- **Row-based:** logs actual before/after row values — verbose but always deterministic. Use in production.

Common pattern: one primary for writes, one or more read replicas behind a load balancer for read-heavy queries.

### Key syntax differences from PostgreSQL

MySQL uses backtick quoting, `AUTO_INCREMENT`, and `ENUM` column types. These differences trip up engineers migrating queries from PostgreSQL.

```sql
-- AUTO_INCREMENT instead of SERIAL / IDENTITY
CREATE TABLE users (
  id         INT AUTO_INCREMENT PRIMARY KEY,
  email      VARCHAR(255) NOT NULL UNIQUE,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Backtick identifier quoting (PostgreSQL uses double-quotes)
SELECT `name`, `email` FROM `users`;

-- ENUM as a column type (PostgreSQL requires a separate CREATE TYPE)
ALTER TABLE orders
  ADD COLUMN status ENUM('pending', 'processing', 'shipped', 'cancelled')
  NOT NULL DEFAULT 'pending';

-- JSON column (MySQL 8.0+) — functional but less capable than PG's JSONB
CREATE TABLE products (metadata JSON);
SELECT JSON_EXTRACT(metadata, '$.color') FROM products;
-- shorthand:  SELECT metadata->>'$.color' FROM products;

-- No partial indexes — use a generated column + regular index instead
ALTER TABLE users
  ADD COLUMN active_email VARCHAR(255) GENERATED ALWAYS AS (IF(active, email, NULL)) STORED;
CREATE INDEX idx_active_email ON users(active_email);
```

### EXPLAIN in MySQL

MySQL's `EXPLAIN` shows the query plan without executing it. The `type` column is the key signal: `ALL` is a full table scan; `eq_ref`/`const` indicates an optimal index lookup.

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 123 AND status = 'pending';

-- Key output columns:
-- type      ALL (full scan) → index → range → ref → eq_ref → const  (worst → best)
-- key       which index MySQL chose (NULL = no index used)
-- rows      estimated rows examined
-- filtered  % of rows that pass the WHERE after index filtering
-- Extra     "Using index" (covering — good), "Using filesort" (expensive), "Using temporary"
```

### MySQL vs PostgreSQL

PostgreSQL offers a richer feature set (JSONB, partial indexes, richer window functions); MySQL wins on ubiquity and ecosystem familiarity in legacy stacks.

| Feature | MySQL | PostgreSQL |
|---|---|---|
| **JSONB** | JSON text column, limited indexing | Binary, indexable, rich operators |
| **Full-text search** | `FULLTEXT` index | `tsvector` / `tsquery` |
| **Transactions** | ACID, MVCC (InnoDB) | ACID, MVCC |
| **Default isolation** | REPEATABLE READ | READ COMMITTED |
| **Window functions** | Full support (8.0+) | Full support |
| **Recursive CTEs** | Yes (8.0+) | Yes |
| **Partial indexes** | No (generated column workaround) | Yes |
| **Extensions** | Limited | Many (PostGIS, pgvector, pg_stat_statements) |
| **Clustered index** | PK is the clustered index | Heap + separate indexes |
| **Replication** | Binlog, GTID, semi-sync | Logical/physical streaming |
| **Best for** | MySQL-invested teams, AWS Aurora | Complex queries, JSONB, GIS, extensions |

**Choosing:** PostgreSQL for greenfield. MySQL when inheriting an existing stack, your team has deep MySQL expertise, or you're using AWS Aurora (MySQL-compatible managed service).

## 19.3 PostgreSQL

PostgreSQL is the most feature-rich open-source relational database and the default for most modern application backends. Its ACID guarantees are strict by default, its index types are varied (B-tree for general equality/range, GIN for JSON and array containment, GiST for geometric and full-text data, partial indexes for filtered subsets), and its support for window functions, CTEs, lateral joins, and full-text search covers a wide range of analytical and application queries. Senior interviews probe `EXPLAIN ANALYZE` because understanding query plans — why a sequential scan is happening instead of an index scan, what the row estimates and actual costs mean, where a Sort or Hash is expensive — is how you diagnose and fix slow queries in production rather than guessing at indexes.

### Indexes

#### B-tree (Default)

```sql
CREATE INDEX idx_users_email ON users(email);

-- Composite index
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- Order matters! This index helps:
SELECT * FROM orders WHERE user_id = 123 ORDER BY created_at;

-- But NOT:
SELECT * FROM orders WHERE created_at > '2024-01-01'; -- created_at not leftmost
```

#### Partial Index

```sql
CREATE INDEX idx_active_users ON users(email) WHERE active = true;

-- Only indexes active users, smaller, faster
SELECT * FROM users WHERE email = 'alice@example.com' AND active = true;
```

#### Covering Index (INCLUDE)

```sql
CREATE INDEX idx_orders_user_total
ON orders(user_id)
INCLUDE (total, created_at);

-- Index contains all needed columns (no table lookup)
SELECT user_id, total, created_at FROM orders WHERE user_id = 123;
```

#### GIN (Generalized Inverted Index) for JSONB

```sql
CREATE INDEX idx_metadata ON products USING GIN (metadata);

-- Fast JSONB queries
SELECT * FROM products WHERE metadata @> '{"color": "red"}';
SELECT * FROM products WHERE metadata ? 'size';
```

#### BRIN (Block Range Index) for Time-Series

```sql
CREATE INDEX idx_events_time ON events USING BRIN (created_at);

-- Very small index, good for ordered data (time-series, logs)
SELECT * FROM events WHERE created_at > '2024-01-01';
```

### EXPLAIN ANALYZE

`EXPLAIN ANALYZE` executes the query and returns actual vs estimated row counts and timings. A large gap between estimated and actual rows signals stale statistics — run `ANALYZE` to refresh them.

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'alice@example.com';

-- Output:
-- Seq Scan on users (cost=0.00..5.25 rows=1 width=40) (actual time=0.012..0.025 rows=1 loops=1)
--   Filter: (email = 'alice@example.com')

-- Add index:
CREATE INDEX idx_users_email ON users(email);

EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'alice@example.com';

-- Output:
-- Index Scan using idx_users_email on users (cost=0.15..8.17 rows=1 width=40) (actual time=0.015..0.016 rows=1 loops=1)
```

**Key metrics:**

- **cost**: Estimated (higher = slower)
- **rows**: Estimated rows
- **actual time**: Real time (ms)
- **Seq Scan**: Full table scan (slow)
- **Index Scan**: Uses index (fast)

### Window Functions

Window functions compute a value for each row using a sliding window of related rows — without collapsing rows like `GROUP BY`. Useful for rankings, running totals, and moving averages over ordered data.

```sql
-- Rank employees by salary within department
SELECT
  name,
  department,
  salary,
  RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank
FROM employees;

-- Running total
SELECT
  date,
  amount,
  SUM(amount) OVER (ORDER BY date) as running_total
FROM transactions;

-- Moving average (last 7 days)
SELECT
  date,
  value,
  AVG(value) OVER (
    ORDER BY date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) as moving_avg_7d
FROM metrics;
```

### CTEs & Recursive Queries

#### CTE (Common Table Expression)

```sql
WITH high_value_orders AS (
  SELECT * FROM orders WHERE total > 1000
)
SELECT
  users.name,
  COUNT(*) as order_count,
  SUM(high_value_orders.total) as total_value
FROM users
JOIN high_value_orders ON users.id = high_value_orders.user_id
GROUP BY users.name;
```

#### Recursive CTE (Hierarchical Data)

```sql
-- Employee hierarchy
WITH RECURSIVE org_chart AS (
  -- Anchor: Top-level managers
  SELECT id, name, manager_id, 1 as level
  FROM employees
  WHERE manager_id IS NULL

  UNION ALL

  -- Recursive: Employees reporting to previous level
  SELECT e.id, e.name, e.manager_id, oc.level + 1
  FROM employees e
  JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart ORDER BY level, name;
```

### Transactions & Isolation Levels

PostgreSQL's default isolation (`READ COMMITTED`) prevents dirty reads but allows non-repeatable reads. Use `SERIALIZABLE` for financial operations or any logic that reads-then-writes the same rows in one transaction.

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;
-- or ROLLBACK;
```

#### Isolation Levels

| Level                           | Dirty Read | Non-repeatable Read | Phantom Read     |
| ------------------------------- | ---------- | ------------------- | ---------------- |
| **READ UNCOMMITTED**            | ✅         | ✅                  | ✅               |
| **READ COMMITTED** (PG default) | ❌         | ✅                  | ✅               |
| **REPEATABLE READ**             | ❌         | ❌                  | ✅ (PG prevents) |
| **SERIALIZABLE**                | ❌         | ❌                  | ❌               |

## 19.4 ORMs

Object-Relational Mappers abstract SQL behind a type-safe API. Prisma generates a typed client from your schema; Drizzle stays close to SQL with a query-builder feel. The right choice depends on team preference — Prisma wins on DX and ecosystem, Drizzle wins on bundle size and flexibility.

### Prisma

```typescript
// schema.prisma
model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
  posts Post[]
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  author    User     @relation(fields: [authorId], references: [id])
  authorId  Int
  createdAt DateTime @default(now())
}

// Generated client
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// Queries
const users = await prisma.user.findMany({
  include: { posts: true },
});

const user = await prisma.user.create({
  data: {
    email: 'alice@example.com',
    name: 'Alice',
    posts: {
      create: { title: 'Hello World' },
    },
  },
});
```

### Drizzle (Lightweight, SQL-like)

```typescript
import { drizzle } from "drizzle-orm/node-postgres";
import { pgTable, serial, text } from "drizzle-orm/pg-core";
import { eq } from "drizzle-orm";

const users = pgTable("users", {
  id: serial("id").primaryKey(),
  name: text("name"),
  email: text("email").unique(),
});

const db = drizzle(pool);

// Queries (SQL-like)
const allUsers = await db.select().from(users);
const user = await db.select().from(users).where(eq(users.id, 1));

await db.insert(users).values({ name: "Alice", email: "alice@example.com" });
```

### NoSQL

## 19.5 MongoDB

MongoDB stores data as BSON documents in schema-optional collections. The document model is natural when your data is hierarchical — rather than joining across tables, you embed related data in a single document and retrieve it in one read. The central design question is **embed vs reference**: embed when the data is always read together and has a bounded size (a post with its comments, up to a few dozen); reference when the sub-document grows unbounded or is accessed independently. Interviewers probe the aggregation pipeline — MongoDB's equivalent of SQL GROUP BY + JOIN + filtering in a sequence of stages — because it's both powerful and easy to write inefficiently without understanding indexes.

### Schema Design

#### Embed vs Reference

```javascript
// Embed (1:1, 1:few, read together)
{
  _id: ObjectId("..."),
  name: "Alice",
  address: {
    street: "123 Main St",
    city: "NYC"
  },
  orders: [
    { orderId: 1, total: 100 },
    { orderId: 2, total: 200 }
  ]
}

// Reference (1:many unbounded, independent updates)
// users collection
{
  _id: ObjectId("..."),
  name: "Alice"
}

// orders collection
{
  _id: ObjectId("..."),
  userId: ObjectId("..."),
  total: 100
}
```

### Aggregation Pipeline

MongoDB's aggregation pipeline is the equivalent of SQL's GROUP BY + JOIN + ORDER BY. Each stage transforms the document stream; stages execute in sequence with one stage's output becoming the next's input.

```javascript
db.orders.aggregate([
  // Stage 1: Filter
  { $match: { status: "completed" } },

  // Stage 2: Join (lookup)
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user",
    },
  },

  // Stage 3: Unwind array
  { $unwind: "$user" },

  // Stage 4: Group
  {
    $group: {
      _id: "$user.name",
      totalOrders: { $sum: 1 },
      totalRevenue: { $sum: "$total" },
    },
  },

  // Stage 5: Sort
  { $sort: { totalRevenue: -1 } },

  // Stage 6: Limit
  { $limit: 10 },
]);
```

## 19.6 Redis

Redis is an in-memory data structure server — most commonly used as a cache, but also as a message broker (pub/sub, streams), session store, rate-limiter, and distributed lock. Its speed comes from keeping all data in RAM with optional persistence (RDB snapshots for point-in-time recovery, AOF for operation logging). The central operational question is **eviction policy**: when memory is full, Redis can evict keys by LRU, LFU, or TTL — misunderstanding this is a common production pitfall. Interviewers often ask candidates to sketch a rate-limiter using a sorted set, or to explain why Redis pub/sub is fire-and-forget while Redis Streams give durable, consumer-group replay.

### Data Structures

#### Strings

```bash
SET user:123:name "Alice"
GET user:123:name

INCR views:article:1  # Atomic increment
INCRBY views:article:1 10

SETEX session:abc 3600 "data"  # Expire in 3600 seconds
```

#### Lists (Queues, Activity Feeds)

```bash
LPUSH queue:emails "email1"
LPUSH queue:emails "email2"
RPOP queue:emails  # "email1" (FIFO)

LPUSH feed:user:123 "post1"
LRANGE feed:user:123 0 9  # Latest 10 posts
```

#### Sets (Tags, Unique Visitors)

```bash
SADD tags:article:1 "tech" "programming"
SMEMBERS tags:article:1

SADD visitors:2024-01-15 "user:123" "user:456"
SCARD visitors:2024-01-15  # Count unique visitors

SINTER tags:article:1 tags:article:2  # Common tags
```

#### Sorted Sets (Leaderboards, Priority Queues)

```bash
ZADD leaderboard 100 "alice"
ZADD leaderboard 200 "bob"
ZADD leaderboard 150 "charlie"

ZRANGE leaderboard 0 -1 WITHSCORES  # All members with scores
ZREVRANGE leaderboard 0 2  # Top 3

ZRANK leaderboard "alice"  # Position (0-indexed)
ZINCRBY leaderboard 50 "alice"
```

#### Hashes (Sessions, User Profiles)

```bash
HSET user:123 name "Alice" email "alice@example.com" age 30
HGET user:123 name
HGETALL user:123

HINCRBY user:123 age 1  # Increment field
```

### Caching Patterns

#### Cache-Aside (Lazy Loading)

```typescript
async function getUser(id: number) {
  const cached = await redis.get(`user:${id}`);
  if (cached) return JSON.parse(cached);

  const user = await db.users.findById(id);
  await redis.setex(`user:${id}`, 3600, JSON.stringify(user));
  return user;
}
```

#### Write-Through

```typescript
async function updateUser(id: number, data: any) {
  await db.users.update(id, data);
  await redis.setex(`user:${id}`, 3600, JSON.stringify(data));
}
```

### Eviction Policies

When Redis hits `maxmemory`, it evicts keys according to the configured policy. `allkeys-lru` is the right choice for a pure cache; `noeviction` is appropriate for a session store where losing data is worse than returning an error.

| Policy           | Behavior                               |
| ---------------- | -------------------------------------- |
| **noeviction**   | Return error when memory limit reached |
| **allkeys-lru**  | Evict least recently used keys         |
| **allkeys-lfu**  | Evict least frequently used keys       |
| **volatile-lru** | Evict LRU keys with TTL                |
| **volatile-ttl** | Evict keys with shortest TTL           |

**Recommended for cache:** `allkeys-lru`

## 19.7 DynamoDB and Single-Table Design

DynamoDB is AWS's fully managed, serverless NoSQL database that delivers consistent single-digit millisecond performance at any scale. Unlike relational databases that you provision and manage, DynamoDB scales automatically and bills per read/write capacity unit (or on-demand). It is the default NoSQL choice for AWS-native architectures.

**Core concepts:**
- **Partition Key (Hash Key):** Determines which physical partition stores the item. All items with the same partition key are co-located. Choose a high-cardinality key (user ID, not gender) for even distribution.
- **Sort Key (Range Key, optional):** Orders items within a partition. Enables range queries (`begins_with`, `between`) within a partition.
- **Primary Key = Partition Key + Sort Key** — uniquely identifies an item.
- **GSI (Global Secondary Index):** An alternate partition+sort key projection. Allows querying by different attributes without duplicating the table.
- **LSI (Local Secondary Index):** Alternate sort key for the same partition key — only useful for range queries on the same partition.

**Single-Table Design philosophy:** In relational databases, you create one table per entity type and JOIN them. DynamoDB does not support joins, so the goal is to design a single table that stores multiple entity types, co-located in a way that answers all your access patterns with a single query.

Design rule: **model your access patterns first, then design your keys around them.**

```typescript
// Example: User, Order, and OrderItem in a single table
// Access patterns:
//   - Get user profile: pk="USER#123", sk="PROFILE"
//   - Get all orders for user: pk="USER#123", sk begins_with "ORDER#"
//   - Get specific order: pk="USER#123", sk="ORDER#2024-01-15#abc"
//   - Get items for an order: pk="ORDER#abc", sk begins_with "ITEM#"

import { DynamoDBClient, PutItemCommand, QueryCommand } from "@aws-sdk/client-dynamodb";
import { marshall, unmarshall } from "@aws-sdk/util-dynamodb";

const client = new DynamoDBClient({ region: "us-east-1" });

// Write a user profile
await client.send(new PutItemCommand({
  TableName: "AppTable",
  Item: marshall({
    pk: "USER#123",           // ← partition key: entity type + ID
    sk: "PROFILE",            // ← sort key: distinguishes item types within the partition
    name: "Alice",
    email: "alice@example.com",
    createdAt: "2024-01-01",
  }),
}));

// Write an order for that user
await client.send(new PutItemCommand({
  TableName: "AppTable",
  Item: marshall({
    pk: "USER#123",               // ← same partition as user profile
    sk: "ORDER#2024-01-15#abc",   // ← sort key with date prefix for range queries
    total: 99.99,
    status: "pending",
  }),
}));

// Query ALL items for a user (profile + all orders) in one round-trip
const { Items } = await client.send(new QueryCommand({
  TableName: "AppTable",
  KeyConditionExpression: "pk = :pk",    // ← all items with this partition key
  ExpressionAttributeValues: marshall({ ":pk": "USER#123" }),
}));

const items = Items?.map(unmarshall);
```

**DynamoDB gotchas:**
- No joins, no aggregations, no ad-hoc queries — design your schema around known access patterns
- Hot partitions (uneven key distribution) throttle performance
- Item size limit: 400KB per item
- Strong vs eventually consistent reads (eventually consistent is 2x cheaper)
- DynamoDB Streams: change data capture stream that can trigger Lambda

## 19.8 Cassandra / Astra DB

Apache Cassandra is a distributed, wide-column NoSQL database designed for massive write throughput with no single point of failure. Originally developed at Facebook, it's widely used for time-series data, IoT, and globally distributed workloads. **Astra DB** is DataStax's managed Cassandra service — Cassandra-compatible with a REST/GraphQL API layer.

**Core model:** Data is organized in tables, but unlike relational tables, Cassandra tables are designed around specific queries. The **partition key** determines which node stores the data. The **clustering key** determines the order of rows within a partition.

**Design rules:**
- One table per query pattern — denormalization is expected and correct
- Model for the queries you need, not for data normalization
- Avoid large partitions (aim for < 100MB per partition)
- Cassandra is NOT suitable for ad-hoc queries, complex joins, or ACID transactions across entities

```sql
-- Create a keyspace (like a database/schema)
CREATE KEYSPACE app WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'datacenter1': 3   -- 3 replicas across datacenter1
};

-- Table for user activity feed
-- Access pattern: "get latest N events for a user"
CREATE TABLE app.user_events (
  user_id   UUID,           -- partition key: all events for a user go to the same node
  event_time TIMESTAMP,     -- clustering key: rows within partition ordered by time DESC
  event_type TEXT,
  payload   TEXT,
  PRIMARY KEY (user_id, event_time)
) WITH CLUSTERING ORDER BY (event_time DESC);

-- Insert (extremely fast — append-only log structure)
INSERT INTO app.user_events (user_id, event_time, event_type, payload)
VALUES (uuid(), toTimestamp(now()), 'page_view', '{"url": "/home"}');

-- Query: latest 50 events for a user
SELECT * FROM app.user_events
WHERE user_id = 550e8400-e29b-41d4-a716-446655440000
LIMIT 50;
```

**Cassandra vs DynamoDB:** Both are distributed NoSQL — Cassandra is self-hosted (or Astra for managed), DynamoDB is AWS-only. Cassandra has tunable consistency (ONE to ALL); DynamoDB has eventual/strong consistency per read. Cassandra is often preferred for time-series and IoT; DynamoDB for AWS-native serverless architectures.

**Consistency levels:**
- `ONE` — fastest, least consistent (one replica responds)
- `QUORUM` — majority of replicas (recommended balance of speed + consistency)
- `ALL` — all replicas must respond (strongest, slowest)

## 19.9 Elasticsearch and SOLR

Elasticsearch is a distributed search and analytics engine built on Apache Lucene. It's used for full-text search, log analytics (ELK/Elastic Stack), product search, and auto-complete. **SOLR** is an older Apache project with similar capabilities but less momentum in modern stacks.

**How it works — the inverted index:** A regular database index maps a row ID to the data in that row. An inverted index maps each word/term to the list of documents containing it. This allows sub-millisecond search across millions of documents.

```
Regular index: document 1 → "The quick brown fox"
Inverted index: "quick" → [doc1, doc7, doc42], "brown" → [doc1, doc12, ...]
```

**Core concepts:**
- **Index:** Like a database table. Contains documents.
- **Document:** A JSON object — like a row. Has a unique `_id`.
- **Field:** A key in the document JSON. Has a **mapping type** (text, keyword, integer, date, geo_point).
- **Mapping:** The schema for an index — field types, analyzers.
- **Analyzer:** How text is tokenized and normalized (lowercase, remove stopwords, stem "running" → "run").
- **Relevance scoring:** BM25 algorithm ranks results by term frequency and document frequency.

```javascript
import { Client } from "@elastic/elasticsearch";

const client = new Client({ node: "http://localhost:9200" });

// Index (store) a document
await client.index({
  index: "products",
  id: "1",
  document: {
    name: "Wireless Noise-Cancelling Headphones",
    description: "Premium over-ear headphones with active noise cancellation",
    price: 299.99,
    category: "electronics",
    inStock: true,
  },
});

// Full-text search with fuzzy matching
const { hits } = await client.search({
  index: "products",
  query: {
    multi_match: {               // ← search across multiple fields
      query: "wireless headpones",  // ← typo: Elasticsearch handles with fuzziness
      fields: ["name^2", "description"],  // ← name weighted 2x more than description
      fuzziness: "AUTO",         // ← handles typos automatically
    },
  },
  filter: [
    { term: { inStock: true } },        // ← exact match on keyword field
    { range: { price: { lte: 400 } } }, // ← filter by price range
  ],
  size: 10,   // ← return top 10 results
});

console.log(hits.hits.map(h => h._source));
```

**Elasticsearch in the ELK Stack:** Elasticsearch is the search/storage layer. Logstash (or Filebeat) ships logs to it. Kibana visualizes and queries logs. AWS's managed version is **Amazon OpenSearch Service** (forked from Elasticsearch 7.10 due to licensing). For new cloud projects, OpenSearch is often preferred.

**When to use Elasticsearch vs a database:**
- Use a database for the canonical source of truth
- Use Elasticsearch as a read-optimized search layer (kept in sync via CDC or events)
- Elasticsearch is eventually consistent and not suitable for financial transactions

### Patterns

## 19.10 Multi-Tenancy Patterns

Multi-tenancy is the ability of a single system to serve multiple customers (tenants) while keeping their data completely isolated. Every SaaS product is multi-tenant. The architecture question is *which isolation model* to use — the choice drives cost, compliance posture, operational complexity, and scale. Senior engineers are expected to know the three models cold, articulate the trade-offs, and know how to implement row-level isolation in PostgreSQL.

### Three isolation models

The isolation model drives cost, compliance posture, and upgrade path. Most SaaS products start with shared schema + RLS and selectively upgrade tenants to stronger isolation.

| Model | Description | Isolation | Cost | Compliance fit |
| --- | --- | --- | --- | --- |
| **Shared schema (RLS)** | All tenants in one DB; `tenant_id` column + row-level security policies | Logical | Lowest | Needs careful policy enforcement; shared infra |
| **Schema-per-tenant** | One database, separate Postgres schema per tenant | Medium | Medium | Easier data portability & right-to-erasure (GDPR) |
| **Database-per-tenant** | Dedicated database per tenant | Strongest | Highest | Easiest for strict compliance (HIPAA, SOC 2, data residency) |

Most SaaS products start with **shared schema + RLS** for cost and simplicity, offer **schema-per-tenant** as a growth tier, and reserve **database-per-tenant** for enterprise customers with regulatory requirements. Design for the upgrade path from day one.

### PostgreSQL Row-Level Security (RLS)

RLS enforces tenant isolation at the database engine — even if application code has a bug and forgets a `WHERE tenant_id = ?`, the policy blocks cross-tenant access.

```sql
-- Enable RLS on the table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy: each row is only visible to the tenant it belongs to
CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Your app sets this at the start of every transaction:
-- SET LOCAL app.current_tenant_id = '<uuid>';
```

```typescript
// NestJS interceptor — sets tenant context before every query
@Injectable()
export class TenantInterceptor implements NestInterceptor {
  constructor(@InjectDataSource() private readonly dataSource: DataSource) {}

  async intercept(ctx: ExecutionContext, next: CallHandler) {
    const tenantId = ctx.switchToHttp().getRequest().user?.tenantId;
    if (!tenantId) throw new UnauthorizedException('No tenant context');

    // SET LOCAL scopes the setting to the current transaction only
    await this.dataSource.query(
      `SET LOCAL app.current_tenant_id = $1`, [tenantId]
    );
    return next.handle();
  }
}
```

**Critical:** use `SET LOCAL` (transaction-scoped), never `SET` (session-scoped). With a connection pool, a session-level setting will leak to the next request that reuses the connection. Also use parameterised queries (`$1`) rather than string interpolation to prevent SQL injection in the tenant ID itself.

### Schema-per-tenant provisioning

Provisioning creates a new Postgres schema and runs migrations scoped to it. This is typically done asynchronously in a background job triggered by signup, not inline in the request.

```typescript
async function provisionTenant(tenantId: string, db: DataSource) {
  const schema = `tenant_${tenantId.replace(/-/g, '_')}`;
  await db.query(`CREATE SCHEMA "${schema}"`);
  // Run all migrations scoped to the new schema
  await runMigrations({ schema });
}
```

Migrations must run against **every schema** on deploy. Tools like Flyway support `flyway.schemas` for this; alternatively, query `information_schema.schemata` to enumerate schemas and loop.

### Tenant context propagation

In every model, the tenant identity must flow through every layer — HTTP request → service → repository → database — without being passed as an explicit function argument everywhere. Two common patterns:

**AsyncLocalStorage (Node.js)** — lightweight, no framework dependency:

```typescript
const tenantStorage = new AsyncLocalStorage<{ tenantId: string }>();

// Set once at the request boundary (middleware)
app.use((req, _res, next) => {
  tenantStorage.run({ tenantId: req.user.tenantId }, next);
});

// Read anywhere in the call stack without prop-drilling
export function getCurrentTenantId(): string {
  return tenantStorage.getStore()!.tenantId;
}
```

**NestJS REQUEST-scoped provider** — integrates with the DI container, automatically tied to the request lifecycle.

**Interview framing:** "How would you implement multi-tenancy in a new SaaS product?" — Start with shared schema + RLS for cost and simplicity; add `tenant_id` as a non-nullable, indexed column on every tenanted table; design the migration path to schema-per-tenant before you need it; never reach for database-per-tenant on day one unless you already have regulatory requirements that demand it.

---

## 19.11 Caching Strategies

Caching operates at **multiple independent layers** in a full-stack system — each with different scopes, TTLs, invalidation mechanisms, and failure modes. A cache miss at the CDN level that causes a full origin render is a very different problem from a Redis cache miss that falls through to a single DB query. Senior engineers need to reason about all layers together, choose the right strategy per layer, and handle the hardest part: cache invalidation. This section maps the full stack from browser to disk.

### The full caching stack

```
Browser Cache
    ↓ miss
CDN / Edge Cache  (CloudFront, Fastly, Vercel Edge)
    ↓ miss
Application Cache  (Redis, in-process LRU)
    ↓ miss
Database Buffer Cache  (PostgreSQL shared_buffers, materialized views)
    ↓ miss
Disk I/O
```

Each layer is faster and cheaper than the one below. The goal is to maximise hit rate at the highest layers.

### Layer 1: Browser cache

Controlled entirely by HTTP response headers:

| Header | Behaviour |
| --- | --- |
| `Cache-Control: max-age=31536000, immutable` | Cache 1 year, never revalidate — use for content-hashed static assets |
| `Cache-Control: no-cache` | Always revalidate with `ETag`/`Last-Modified` before using cached copy |
| `Cache-Control: no-store` | Never cache — for sensitive data, auth responses |
| `Cache-Control: stale-while-revalidate=60` | Serve stale for up to 60 s while revalidating in background |
| `ETag: "abc123"` | Client sends `If-None-Match`; server responds `304 Not Modified` if unchanged |

```typescript
// Express — aggressive caching for hashed static assets
app.use('/static', express.static('dist', { maxAge: '1y', immutable: true }));

// Short TTL with stale-while-revalidate for API responses
res.set('Cache-Control', 'public, max-age=60, stale-while-revalidate=30');
res.set('ETag', generateETag(responseBody));
```

### Layer 2: CDN / edge cache

CDNs cache responses at PoPs close to the user. The two most important CDN-specific features are **surrogate keys** (cache tags) and **stale-while-revalidate at the edge**.

**Surrogate keys** let you tag cached responses and purge a group atomically:

```typescript
// Tag the response — Fastly and Cloudflare support this header
res.set('Surrogate-Key', `user-${userId} users-list`);
res.set('Surrogate-Control', 'max-age=3600');

// Purge all cached user responses when a user is updated
await fastly.purgeByTag(`user-${userId}`);
```

**Stale-while-revalidate at the CDN:** the edge serves the cached response immediately and asynchronously refreshes from origin. Users always get a response; the origin sees ~1 request per PoP per TTL, not one per user.

### Layer 3: Application cache (Redis)

See §19.6 for Redis data structures. The caching patterns in context:

| Pattern | Description | When to use |
| --- | --- | --- |
| **Cache-aside** | App reads Redis first; on miss, reads DB and writes to Redis | Most read-heavy cases — simplest |
| **Write-through** | Write to Redis and DB simultaneously on every mutation | Low read latency after writes; consistent |
| **Write-behind** | Write to Redis immediately; persist to DB asynchronously | Very high write throughput; tolerate eventual durability |
| **Read-through** | Cache intercepts all reads; populates itself on miss | Managed caches (ElastiCache, DAX) |

**Cache stampede prevention.** When a hot key expires, many requests simultaneously miss and all hammer the DB. Prevent with a mutex lock on cache miss:

```typescript
async function getWithLock<T>(
  key: string,
  ttl: number,
  fetch: () => Promise<T>
): Promise<T> {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const lockKey = `lock:${key}`;
  const acquired = await redis.set(lockKey, '1', 'NX', 'EX', 5); // 5 s TTL
  if (!acquired) {
    await new Promise(r => setTimeout(r, 100));  // brief backoff, then retry
    return getWithLock(key, ttl, fetch);
  }
  try {
    const data = await fetch();
    await redis.setex(key, ttl, JSON.stringify(data));
    return data;
  } finally {
    await redis.del(lockKey);
  }
}
```

### Layer 4: Database buffer cache

PostgreSQL doesn't have a query result cache (MySQL's was removed in 8.0 too). What it does have:

- **`shared_buffers`** — in-memory buffer pool for recently accessed data pages. Default 128 MB; recommended 25% of RAM. Eliminates disk I/O for hot data without any application code change.
- **Connection pooling (PgBouncer)** — reuses database connections, eliminating TCP + auth handshake overhead on every request. Run in transaction mode for most workloads.
- **Materialized views** — pre-compute expensive aggregations:

```sql
CREATE MATERIALIZED VIEW monthly_revenue AS
  SELECT date_trunc('month', created_at) AS month, SUM(amount) AS revenue
  FROM orders GROUP BY 1;

-- Refresh without locking reads (requires a unique index)
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;
```

### Cache invalidation patterns

The right pattern depends on how much staleness is tolerable. TTL is simplest; event-driven gives the freshest results at the cost of coordination infrastructure.

| Pattern | How | Trade-off |
| --- | --- | --- |
| **TTL expiry** | Entry expires after a fixed time | Simple; allows staleness up to TTL |
| **Write-through** | Update cache on every write | Always consistent; increases write latency |
| **Event-driven** | Publish event on mutation; subscriber deletes affected keys | Fresh; requires coordination infrastructure |
| **CDN surrogate key purge** | Call CDN purge API after mutation | Immediate; CDN API call adds write latency |
| **Asset hash busting** | New filename on every build | Instant client-side invalidation; requires build integration |

**The fundamental trade-off:** freshness vs performance. The more aggressively you cache, the more you risk serving stale data. The right TTL always depends on the data's tolerance for staleness: session tokens — no caching; user profile — 60 s fine; static product copy — hours or days fine.

**Interview framing:** "How would you design caching for a read-heavy e-commerce product page?" — Walk the stack: immutable CDN caching for static assets (1 year + hash-busted on deploy), short TTL with `stale-while-revalidate` at the edge for product API responses (60 s SWR), Redis cache-aside for DB query results (5–15 min TTL), event-driven invalidation on product update (publish `product.updated` → subscriber clears Redis key + CDN surrogate purge). The follow-up: "What happens when the price changes?" — that's the invalidation question, answered by event-driven cache clearing.

## Database Priority Summary

| Topic                                                       | Priority                     |
| ----------------------------------------------------------- | ---------------------------- |
| **SQL Fundamentals**                                        |                              |
| JOINs (INNER, LEFT, RIGHT, FULL, self)                      | **Critical**                 |
| ACID properties                                             | **Critical**                 |
| Transaction isolation levels                                | **Deep**                     |
| **PostgreSQL**                                              |                              |
| Indexes (B-tree, partial, covering, GIN, BRIN)              | **Critical**                 |
| EXPLAIN ANALYZE                                             | **Deep**                     |
| Window functions                                            | **Deep**                     |
| CTEs (recursive)                                            | **Deep**                     |
| Transactions, isolation levels                              | **Deep**                     |
| JSONB                                                       | **Important**                |
| **MySQL**                                                   |                              |
| InnoDB storage engine, clustered index                      | **Important**                |
| Replication (binlog, GTID, row-based)                       | **Important**                |
| PostgreSQL vs MySQL tradeoffs                               | **Learn**                    |
| **ORMs**                                                    |                              |
| Prisma                                                      | **Important** (most popular) |
| Drizzle                                                     | **Learn** (fast growing)     |
| **MongoDB**                                                 |                              |
| Embed vs Reference                                          | **Important**                |
| Aggregation pipeline                                        | **Important**                |
| **Redis**                                                   |                              |
| Data structures (Strings, Lists, Sets, Sorted Sets, Hashes) | **Critical**                 |
| Caching patterns (cache-aside, write-through)               | **Critical**                 |
| Eviction policies                                           | **Deep**                     |
| **DynamoDB**                                                |                              |
| Partition key, sort key, GSI                                | **Deep**                     |
| Single-table design                                         | **Important**                |
| DynamoDB Streams, DAX                                       | **Know**                     |
| **Cassandra / Astra DB**                                    |                              |
| Partition key + clustering key design                       | **Learn**                    |
| Consistency levels (ONE/QUORUM/ALL)                         | **Learn**                    |
| **Elasticsearch**                                           |                              |
| Inverted index, analyzers, relevance scoring                | **Learn**                    |
| ELK Stack / OpenSearch                                      | **Learn**                    |
| **Multi-Tenancy**                                           |                              |
| Three isolation models (RLS, schema, DB)                    | **Deep**                     |
| PostgreSQL RLS implementation                               | **Important**                |
| Tenant context via AsyncLocalStorage                        | **Important**                |
| **Caching Strategies**                                      |                              |
| Full cache stack (browser → CDN → Redis → DB)               | **Deep**                     |
| HTTP caching headers (Cache-Control, ETag, SWR)             | **Important**                |
| CDN surrogate keys / cache purge                            | **Important**                |
| Cache stampede prevention (mutex lock)                      | **Learn**                    |
| Cache invalidation patterns                                 | **Deep**                     |

---


# 20. Design Principles

SOLID, GRASP, and GoF design patterns are the vocabulary of professional software design. Interviewers ask about them to assess whether you write code that can be maintained, extended, and tested by a team — not just code that works today. These aren't academic concepts: they appear constantly in code reviews, architecture discussions, and system design interviews.

## 20.1 SOLID

SOLID is a set of five principles for writing maintainable object-oriented code, coined by Robert Martin. Each principle addresses a specific way that code becomes hard to change over time.

### Single Responsibility Principle (SRP)

A class or module should have **one and only one reason to change**. If a class handles data persistence, sends emails, and logs events, then a change to the logging system forces you to touch the same class as a change to the email template. These concerns should live in separate classes.

**Why it matters:** SRP is the most commonly violated principle. Each added responsibility to an existing class creates coupling that makes testing harder and changes riskier.

```typescript
// ❌ Bad: Multiple responsibilities
class UserService {
  async createUser(data: CreateUserDto) {
    const user = await this.db.users.create(data);
    await this.sendWelcomeEmail(user.email); // Email responsibility
    await this.logger.log("User created"); // Logging responsibility
    return user;
  }
}

// ✅ Good: Single responsibility
class UserService {
  constructor(
    private db: Database,
    private emailService: EmailService,
    private logger: Logger,
  ) {}

  async createUser(data: CreateUserDto) {
    const user = await this.db.users.create(data);
    await this.emailService.sendWelcome(user.email);
    this.logger.log("User created", { userId: user.id });
    return user;
  }
}
```

### Open/Closed Principle (OCP)

Software entities should be **open for extension but closed for modification**. When adding a new feature requires editing an existing class, you risk breaking existing behaviour. The fix is to design for extension via interfaces and polymorphism so new functionality can be added as new code, not edited code.

**Why it matters:** Every time you add an `if/else` or `case` to an existing function to handle a new type, you're violating OCP. The `switch` growing to handle Stripe, PayPal, Apple Pay... signals a design problem.

```typescript
// ❌ Bad: Must modify class to add payment method
class PaymentProcessor {
  process(amount: number, method: string) {
    if (method === "stripe") {
      // Stripe logic
    } else if (method === "paypal") {
      // PayPal logic
    }
  }
}

// ✅ Good: Open for extension, closed for modification
interface PaymentGateway {
  process(amount: number): Promise<void>;
}

class StripeGateway implements PaymentGateway {
  async process(amount: number) {
    // Stripe logic
  }
}

class PaymentProcessor {
  constructor(private gateway: PaymentGateway) {}

  async process(amount: number) {
    return this.gateway.process(amount);
  }
}
```

### Liskov Substitution Principle (LSP)

Subtypes must be substitutable for their base types without breaking the program's correctness. In other words, if `S` extends `T`, then anywhere you use a `T`, you should be able to use an `S` without unexpected behaviour.

**Why it matters:** LSP violations indicate forced inheritance — the subclass doesn't truly IS-A the parent. The fix is usually composition over inheritance.

```typescript
// ❌ Bad: Square overrides Rectangle in a way that breaks the contract
class Rectangle {
  constructor(protected width: number, protected height: number) {}
  setWidth(w: number)  { this.width = w; }
  setHeight(h: number) { this.height = h; }
  area() { return this.width * this.height; }
}

class Square extends Rectangle {
  // Square forces width === height, violating Rectangle's contract
  setWidth(w: number)  { this.width = w; this.height = w; }   // ← side effect!
  setHeight(h: number) { this.width = h; this.height = h; }   // ← side effect!
}

function resizeAndPrint(rect: Rectangle) {
  rect.setWidth(5);
  rect.setHeight(10);
  console.log(rect.area()); // Expects 50, but Square gives 100 — LSP violated!
}

// ✅ Good: Don't force inheritance when IS-A is wrong
interface Shape {
  area(): number;
}
class Rectangle implements Shape {
  constructor(private w: number, private h: number) {}
  area() { return this.w * this.h; }
}
class Square implements Shape {
  constructor(private side: number) {}
  area() { return this.side * this.side; }
}
```

### Interface Segregation Principle (ISP)

No client should be forced to depend on methods it doesn't use. Large "fat" interfaces force implementors to provide stub implementations for methods they don't need, which is a sign the interface is doing too much.

**Why it matters:** ISP keeps interfaces focused. It also means changes to one part of a large interface don't ripple to unrelated implementations.

```typescript
// ❌ Bad: One fat interface forces Robot to implement eat() and sleep()
interface Worker {
  work(): void;
  eat(): void;    // ← robots don't eat
  sleep(): void;  // ← robots don't sleep
}

class Robot implements Worker {
  work() { /* ... */ }
  eat() { throw new Error("Robots don't eat!"); }   // ← forced stub
  sleep() { throw new Error("Robots don't sleep!"); } // ← forced stub
}

// ✅ Good: Split into focused interfaces
interface Workable  { work(): void; }
interface Feedable  { eat(): void; }
interface Restable  { sleep(): void; }

class HumanWorker implements Workable, Feedable, Restable {
  work()  { /* ... */ }
  eat()   { /* ... */ }
  sleep() { /* ... */ }
}

class Robot implements Workable {
  work() { /* ... */ }  // ← only implements what it actually does
}
```

### Dependency Inversion Principle (DIP)

High-level modules should not depend on low-level modules. Both should depend on abstractions. And abstractions should not depend on details — details should depend on abstractions.

**Why it matters:** DIP is the foundation of testable code. When a service creates its own dependencies (`new Database()`), you can't swap them in tests. When it receives them as abstractions (interfaces), you can inject mocks freely.

```typescript
// ❌ Bad: High-level module creates a low-level dependency directly
class UserService {
  private db = new MySQLDatabase();   // ← tight coupling, impossible to test without MySQL

  getUser(id: number) {
    return this.db.query(`SELECT * FROM users WHERE id = ${id}`);
  }
}

// ✅ Good: Both depend on the abstraction (interface)
interface IDatabase {
  query(sql: string, params?: unknown[]): Promise<unknown[]>;
}

class UserService {
  constructor(private db: IDatabase) {}  // ← receives abstraction, caller provides implementation

  getUser(id: number) {
    return this.db.query("SELECT * FROM users WHERE id = $1", [id]);
  }
}

// In tests: inject a mock
const mockDb: IDatabase = { query: vi.fn().mockResolvedValue([{ id: 1 }]) };
const service = new UserService(mockDb);   // ← fully testable

// In production: inject the real implementation
const realService = new UserService(new PostgresDatabase());
```

## 20.2 GRASP Patterns

GRASP (General Responsibility Assignment Software Patterns) are 9 principles from Craig Larman's _Applying UML and Patterns_ for deciding which class should own which responsibility. They're less famous than GoF but equally practical.

| Pattern | Rule |
|---|---|
| **Information Expert** | Assign a responsibility to the class that has the information needed to fulfil it. A `CartItem` knows its own price, so it should calculate its own line total. |
| **Creator** | Assign class B the responsibility to create instances of A if B contains, aggregates, or closely uses A. `Order` creates `OrderItem`s. |
| **Controller** | Assign handling of a system event to a dedicated class (use-case controller or facade). `CheckoutController` handles the checkout system event — not `Order` or `User`. |
| **Low Coupling** | Assign responsibilities to minimize dependencies between classes. Fewer dependencies = easier to change one class without breaking others. |
| **High Cohesion** | Keep related responsibilities together. A class that does 10 unrelated things has low cohesion — split it. |
| **Polymorphism** | When behaviour varies by type, use interfaces/abstract methods rather than if/switch chains. |
| **Pure Fabrication** | Create a class that doesn't represent a domain concept but reduces coupling (e.g., `UserRepository` — there's no "repository" in the real world, but it separates data access from domain logic). |
| **Indirection** | Introduce a mediator between components to reduce direct coupling. A `PaymentGateway` interface mediates between `OrderService` and the Stripe SDK. |
| **Protected Variations** | Identify points of variation and wrap them behind a stable interface so they can change without ripple effects. |

## 20.3 GoF Design Patterns

The Gang of Four (Gamma, Helm, Johnson, Vlissides) catalogued 23 design patterns in _Design Patterns: Elements of Reusable Object-Oriented Software_ (1994). They fall into three categories.

### Creational Patterns

**Singleton**

Ensures only one instance of a class exists. Useful for shared resources (database connection pool, config).

⚠️ Overuse creates hidden global state that makes testing harder — prefer dependency injection where possible.

```typescript
class DatabaseConnection {
  private static instance: DatabaseConnection;
  private constructor() {}   // ← private: prevents external `new`

  static getInstance(): DatabaseConnection {
    if (!DatabaseConnection.instance) {
      DatabaseConnection.instance = new DatabaseConnection();
    }
    return DatabaseConnection.instance;
  }
}
```

**Factory Method**

Define an interface for creating an object, but let subclasses or a factory function decide which concrete class to instantiate.

```typescript
interface Logger { log(message: string): void; }

class ConsoleLogger implements Logger {
  log(msg: string) { console.log(msg); }
}
class FileLogger implements Logger {
  log(msg: string) { fs.appendFileSync("app.log", msg + "\n"); }
}

// Factory: caller asks for a Logger, doesn't know the concrete type
function createLogger(env: string): Logger {
  return env === "production" ? new FileLogger() : new ConsoleLogger();
}
```

**Builder**

Construct complex objects step by step. Particularly useful when an object has many optional fields or when construction order matters.

```typescript
class QueryBuilder {
  private table = "";
  private conditions: string[] = [];
  private limit?: number;

  from(table: string) { this.table = table; return this; }      // ← fluent interface
  where(condition: string) { this.conditions.push(condition); return this; }
  take(n: number) { this.limit = n; return this; }

  build(): string {
    let sql = `SELECT * FROM ${this.table}`;
    if (this.conditions.length) sql += ` WHERE ${this.conditions.join(" AND ")}`;
    if (this.limit) sql += ` LIMIT ${this.limit}`;
    return sql;
  }
}

const query = new QueryBuilder()
  .from("users")
  .where("active = true")
  .where("age > 18")
  .take(10)
  .build();
// "SELECT * FROM users WHERE active = true AND age > 18 LIMIT 10"
```

### Structural Patterns

**Adapter**

Wraps an incompatible interface in a compatible one. The most common use: adapting third-party SDKs to match your application's interfaces.

```typescript
// Your app's interface
interface IPaymentGateway {
  charge(amountCents: number, currency: string): Promise<{ transactionId: string }>;
}

// Third-party Stripe SDK (different interface)
class StripeSdk {
  createPaymentIntent(params: { amount: number; currency: string }) { /* ... */ }
}

// Adapter: wraps Stripe to match IPaymentGateway
class StripeAdapter implements IPaymentGateway {
  constructor(private stripe: StripeSdk) {}

  async charge(amountCents: number, currency: string) {
    const intent = await this.stripe.createPaymentIntent({ amount: amountCents, currency });
    return { transactionId: intent.id };
  }
}

// Your OrderService only knows about IPaymentGateway — not Stripe specifics
```

**Decorator**

Adds behaviour to an object dynamically by wrapping it, without modifying the original class. TypeScript's `@Decorator` syntax uses this pattern.

```typescript
interface IUserService { getUser(id: number): Promise<User>; }

class UserService implements IUserService {
  async getUser(id: number) { /* fetch from DB */ }
}

// Logging decorator: wraps UserService without modifying it
class LoggingUserService implements IUserService {
  constructor(private inner: IUserService) {}

  async getUser(id: number) {
    console.log(`getUser(${id}) called`);
    const result = await this.inner.getUser(id);  // ← delegates to wrapped service
    console.log(`getUser(${id}) returned:`, result);
    return result;
  }
}

const service = new LoggingUserService(new UserService());
```

**Facade**

Provides a simplified interface over a complex subsystem. Callers interact with the facade — they don't need to know about the underlying complexity.

```typescript
// Complex subsystem: multiple services
class EmailService    { send(to: string, body: string) {} }
class SmsService      { send(to: string, msg: string) {}  }
class PushService     { send(deviceId: string, msg: string) {} }

// Facade: simple unified interface
class NotificationFacade {
  constructor(
    private email: EmailService,
    private sms: SmsService,
    private push: PushService,
  ) {}

  async notifyUser(user: User, message: string) {
    await this.email.send(user.email, message);
    if (user.phone)    await this.sms.send(user.phone, message);
    if (user.deviceId) await this.push.send(user.deviceId, message);
  }
}
// Caller: facade.notifyUser(user, "Your order shipped!")
// No need to know about Email, SMS, Push services directly
```

**Proxy**

Controls access to an object by wrapping it. Use cases: caching, lazy initialisation, access control, logging.

```typescript
interface IUserRepository { findById(id: number): Promise<User | null>; }

// Caching Proxy
class CachingUserRepository implements IUserRepository {
  private cache = new Map<number, User>();

  constructor(private inner: IUserRepository) {}

  async findById(id: number) {
    if (this.cache.has(id)) {
      return this.cache.get(id)!;   // ← return cached value, skip DB
    }
    const user = await this.inner.findById(id);
    if (user) this.cache.set(id, user);
    return user;
  }
}
```

### Behavioural Patterns

**Observer**

An object (subject/publisher) maintains a list of dependents (observers/subscribers) and notifies them of state changes. This is the pub/sub pattern.

```typescript
type EventHandler<T> = (data: T) => void;

class EventEmitter<Events extends Record<string, unknown>> {
  private handlers = new Map<keyof Events, Set<EventHandler<unknown>>>();

  on<K extends keyof Events>(event: K, handler: EventHandler<Events[K]>) {
    if (!this.handlers.has(event)) this.handlers.set(event, new Set());
    this.handlers.get(event)!.add(handler as EventHandler<unknown>);
  }

  emit<K extends keyof Events>(event: K, data: Events[K]) {
    this.handlers.get(event)?.forEach(h => h(data));
  }
}
// Node.js EventEmitter, React state updates, and DOM events all use this pattern
```

**Strategy**

Defines a family of algorithms, encapsulates each one, and makes them interchangeable at runtime.

```typescript
interface SortStrategy { sort(data: number[]): number[]; }

class QuickSort  implements SortStrategy { sort(d: number[]) { /* ... */ return d; } }
class MergeSort  implements SortStrategy { sort(d: number[]) { /* ... */ return d; } }

class Sorter {
  constructor(private strategy: SortStrategy) {}
  setStrategy(s: SortStrategy) { this.strategy = s; }  // ← swap at runtime
  sort(data: number[]) { return this.strategy.sort(data); }
}
```

**Chain of Responsibility**

Pass a request along a chain of handlers. Each handler decides whether to process it or pass it along. **Express middleware is Chain of Responsibility.**

```typescript
type Middleware = (req: Request, res: Response, next: () => void) => void;

// Each middleware either handles the request or calls next()
const authMiddleware: Middleware = (req, res, next) => {
  if (!req.headers.authorization) { res.status(401).send("Unauthorized"); return; }
  next();   // ← pass to next handler in chain
};

const loggingMiddleware: Middleware = (req, res, next) => {
  console.log(`${req.method} ${req.path}`);
  next();
};
```

**Command**

Encapsulates an action as an object, enabling undo/redo, queuing, and logging of operations.

```typescript
interface Command { execute(): void; undo(): void; }

class MoveCommand implements Command {
  constructor(
    private element: Element,
    private dx: number, private dy: number,
  ) {}
  execute() { /* move element by dx, dy */ }
  undo()    { /* move element by -dx, -dy */ }
}

class CommandHistory {
  private history: Command[] = [];
  execute(cmd: Command) { cmd.execute(); this.history.push(cmd); }
  undo()               { this.history.pop()?.undo(); }
}
```

**State**

Object changes its behaviour based on internal state. Instead of large if/switch chains, each state is a class.

```typescript
interface OrderState { process(): void; cancel(): void; }

class PendingState  implements OrderState {
  process() { order.state = new ProcessingState(); }
  cancel()  { order.state = new CancelledState(); }
}
class ProcessingState implements OrderState {
  process() { throw new Error("Already processing"); }
  cancel()  { order.state = new CancelledState(); /* trigger refund */ }
}
// Order delegates to its current state: order.state.process()
```

**Template Method**

Defines the skeleton of an algorithm in a base class, deferring specific steps to subclasses.

```typescript
abstract class DataImporter {
  // Template method — the skeleton
  async import(source: string) {
    const raw = await this.fetch(source);    // ← step 1: fetch
    const parsed = this.parse(raw);          // ← step 2: parse (abstract)
    await this.validate(parsed);             // ← step 3: validate (abstract)
    await this.save(parsed);                 // ← step 4: save
  }

  protected abstract parse(raw: string): unknown[];
  protected abstract validate(data: unknown[]): Promise<void>;

  // Shared implementations
  protected async fetch(source: string) { return fs.readFile(source, "utf8"); }
  protected async save(data: unknown[]) { /* default DB save */ }
}

class CsvImporter extends DataImporter {
  protected parse(raw: string)              { return raw.split("\n").map(parseCSVRow); }
  protected async validate(data: unknown[]) { /* CSV-specific validation */ }
}
```

## 20.4 Layered Architecture

Architecture patterns describe how to organize the components of an application. The goal in all cases is the same: **separate concerns so that each part can change independently**.

### 3-Layer Architecture (Most Common)

The classic backend structure: every request flows down through three layers, and data flows back up.

```
┌─────────────────────────────┐
│   Presentation Layer         │  ← HTTP controllers, route handlers, DTOs
│   (Routes / Controllers)     │    Validates input, delegates to service layer
├─────────────────────────────┤
│   Business / Application     │  ← Services, use-case classes
│   Layer                      │    Business logic lives here — no HTTP, no SQL
├─────────────────────────────┤
│   Data Access Layer          │  ← Repositories, ORMs, raw SQL
│   (Repositories)             │    Translates between business objects and storage
└─────────────────────────────┘
```

```typescript
// 3-layer directory structure for a NestJS/Express API:
// src/
//   controllers/     ← Presentation: UserController (handles HTTP)
//   services/        ← Business: UserService (business rules)
//   repositories/    ← Data: UserRepository (SQL/ORM)
//   dto/             ← Data Transfer Objects (request/response shapes)
//   entities/        ← Domain entities (User, Order)

// Controller: only handles HTTP concerns
@Controller("/users")
class UserController {
  constructor(private userService: UserService) {}

  @Get("/:id")
  async getUser(@Param("id") id: string) {
    return this.userService.getUserById(Number(id));   // ← delegates immediately
  }
}

// Service: only handles business logic (no HTTP, no SQL)
class UserService {
  constructor(private userRepo: UserRepository) {}

  async getUserById(id: number): Promise<UserDto> {
    const user = await this.userRepo.findById(id);
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return toUserDto(user);   // ← transforms domain entity to DTO
  }
}

// Repository: only handles data access
class UserRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: number): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { id } });
  }
}
```

### Hexagonal Architecture (Ports & Adapters)

Hexagonal architecture (also called Ports & Adapters, coined by Alistair Cockburn) takes the 3-layer model further: the **domain/core** has zero dependencies on external systems. Instead, it defines **Ports** (interfaces) describing what it needs. **Adapters** implement those ports for specific technologies.

```
                        ┌─────────────────────┐
   HTTP Request ──►  ┤  ←  Input Adapter       ├  ←  The domain doesn't know
   CLI command ──►   ┤  (REST Controller,       ├     about HTTP or CLI
                     ┤   GraphQL resolver)      ├
                     ├─────────────────────────┤
                     │   APPLICATION DOMAIN    │  ← Pure business logic
                     │   (Use Cases / Services)│    No framework imports
                     │   IUserRepo (Port) ──►  │
                     ├─────────────────────────┤
   PostgreSQL  ◄──  ┤  →  Output Adapter       ├  →  The domain doesn't know
   Kafka msg   ◄──  ┤  (PrismaUserRepo,        ├     about Postgres or Kafka
                     ┤   KafkaMessageBus)       ├
                     └─────────────────────────┘
```

```typescript
// Domain defines what it needs via interfaces (Ports)
interface IUserRepository {    // ← Output Port
  findById(id: number): Promise<User | null>;
  save(user: User): Promise<void>;
}

// Domain use case — zero external imports
class RegisterUserUseCase {
  constructor(private repo: IUserRepository) {}

  async execute(cmd: RegisterUserCommand): Promise<void> {
    if (await this.repo.findById(cmd.userId)) throw new Error("Already exists");
    await this.repo.save(new User(cmd.userId, cmd.email));
  }
}

// Output Adapter: implements the Port for Prisma
class PrismaUserRepository implements IUserRepository {
  constructor(private prisma: PrismaClient) {}
  async findById(id: number) { return this.prisma.user.findUnique({ where: { id } }); }
  async save(user: User)     { return this.prisma.user.create({ data: user }); }
}

// Input Adapter: Express controller wires HTTP into the use case
app.post("/users", async (req, res) => {
  await registerUserUseCase.execute(req.body);
  res.status(201).send();
});
```

**Benefits of Hexagonal:** You can swap PostgreSQL for DynamoDB without touching domain logic. You can test domain logic with in-memory repository stubs. The domain is the most important code — it has the fewest dependencies.

## 20.5 Enterprise and Resilience Patterns

These patterns address concerns specific to distributed systems and enterprise applications — they don't appear in the classic GoF catalogue but are equally important for senior engineering interviews.

### Singleton

Ensures only one instance of a class exists in the process. Commonly used for database connections, config managers, and loggers where multiple instances would be wasteful or incorrect.

```typescript
class DatabaseConnection {
  private static instance: DatabaseConnection;
  private constructor() {}

  static getInstance(): DatabaseConnection {
    if (!DatabaseConnection.instance) {
      DatabaseConnection.instance = new DatabaseConnection();
    }
    return DatabaseConnection.instance;
  }
}
```

### Factory

A factory method abstracts object creation behind an interface, letting callers request an instance by type without depending on the concrete class — making it easy to swap implementations.

```typescript
interface PaymentGateway {
  process(amount: number): Promise<void>;
}

class PaymentGatewayFactory {
  create(type: "stripe" | "paypal"): PaymentGateway {
    switch (type) {
      case "stripe":
        return new StripeGateway();
      case "paypal":
        return new PayPalGateway();
      default:
        throw new Error("Unknown gateway");
    }
  }
}
```

### Strategy

The Strategy pattern encapsulates interchangeable algorithms behind a common interface. The caller switches behavior at runtime by swapping the injected strategy — without modifying the host class.

```typescript
interface SortStrategy {
  sort(data: number[]): number[];
}

class QuickSort implements SortStrategy {
  sort(data: number[]) {
    /* ... */
  }
}

class Sorter {
  constructor(private strategy: SortStrategy) {}

  sort(data: number[]) {
    return this.strategy.sort(data);
  }
}
```

### Repository Pattern

The Repository pattern abstracts data access behind an interface, decoupling business logic from the specific ORM or database. Services depend on the interface — making them unit-testable by injecting a mock repository.

```typescript
interface UserRepository {
  findById(id: number): Promise<User | null>;
  findAll(): Promise<User[]>;
  create(data: CreateUserDto): Promise<User>;
}

class PrismaUserRepository implements UserRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: number) {
    return this.prisma.user.findUnique({ where: { id } });
  }
}

// Business logic depends on interface
class UserService {
  constructor(private repo: UserRepository) {}

  async getUser(id: number) {
    return this.repo.findById(id);
  }
}
```

### Circuit Breaker

A circuit breaker wraps calls to a downstream dependency and stops forwarding requests when failure rate exceeds a threshold (OPEN state), preventing cascade failures. After a timeout it probes with a single request (HALF_OPEN) to check recovery.

```typescript
enum CircuitState {
  CLOSED, // Normal operation
  OPEN, // Failing, reject requests
  HALF_OPEN, // Testing if recovered
}

class CircuitBreaker {
  private state = CircuitState.CLOSED;
  private failureCount = 0;
  private successCount = 0;
  private nextAttempt = Date.now();

  constructor(
    private failureThreshold = 5,
    private timeout = 60000,
    private successThreshold = 2,
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      if (Date.now() < this.nextAttempt) {
        throw new Error("Circuit breaker is OPEN");
      }
      this.state = CircuitState.HALF_OPEN;
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  private onSuccess() {
    this.failureCount = 0;

    if (this.state === CircuitState.HALF_OPEN) {
      this.successCount++;
      if (this.successCount >= this.successThreshold) {
        this.state = CircuitState.CLOSED;
        this.successCount = 0;
      }
    }
  }

  private onFailure() {
    this.failureCount++;
    this.successCount = 0;

    if (this.failureCount >= this.failureThreshold) {
      this.state = CircuitState.OPEN;
      this.nextAttempt = Date.now() + this.timeout;
    }
  }
}
```

### Saga Pattern (Orchestration)

The Saga pattern manages multi-step distributed transactions without two-phase commit. Each step calls a service; if a step fails, compensating transactions run in reverse to undo completed steps.

```typescript
class OrderSaga {
  async execute(orderData: CreateOrderDto) {
    let orderId: number;
    let reservationId: number;
    let paymentId: number;

    try {
      // Step 1: Create order
      orderId = await this.orderService.create(orderData);

      // Step 2: Reserve inventory
      reservationId = await this.inventoryService.reserve(orderData.items);

      // Step 3: Process payment
      paymentId = await this.paymentService.charge(orderData.total);

      // Step 4: Confirm order
      await this.orderService.confirm(orderId);

      return { orderId, success: true };
    } catch (err) {
      // Compensating transactions (rollback)
      if (paymentId) await this.paymentService.refund(paymentId);
      if (reservationId) await this.inventoryService.release(reservationId);
      if (orderId) await this.orderService.cancel(orderId);

      return { success: false, error: err.message };
    }
  }
}
```

### Bulkhead

The bulkhead pattern limits the number of **concurrent calls** to a single dependency, preventing a slow downstream service from exhausting shared thread/connection-pool resources and degrading unrelated operations. Named after the watertight compartments that prevent a hull breach from sinking an entire ship.

```typescript
class Bulkhead {
  private inFlight = 0;

  constructor(private readonly maxConcurrent: number) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.inFlight >= this.maxConcurrent) {
      throw new Error(`Bulkhead full (${this.maxConcurrent} in flight)`);
    }
    this.inFlight++;
    try {
      return await fn();
    } finally {
      this.inFlight--;
    }
  }
}

// Payment gateway limited to 10 concurrent calls — a slow provider
// can't starve the connection pool for order, inventory, or user services
const paymentBulkhead = new Bulkhead(10);

async function chargeCard(orderId: string) {
  return paymentBulkhead.execute(() => paymentService.charge(orderId));
}
```

### Timeout

Every call to an external dependency **must have a deadline**. Without one, a hanging downstream call holds a connection open indefinitely, exhausting the pool and cascading failures across the system.

```typescript
function withTimeout<T>(promise: Promise<T>, ms: number, label: string): Promise<T> {
  const deadline = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error(`${label} timed out after ${ms}ms`)), ms)
  );
  return Promise.race([promise, deadline]);
}

// Never await an unconstrained external call
const user = await withTimeout(userService.findById(userId), 500, 'userService.findById');
```

With `fetch`, integrate via `AbortController`:

```typescript
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 500);
try {
  const res = await fetch(url, { signal: controller.signal });
  // ...
} finally {
  clearTimeout(timeoutId);
}
```

### Fallback

When a dependency fails — whether due to a timeout, circuit breaker trip, or network error — return a **safe degraded response** rather than propagating the error up to the user.

```typescript
async function getRecommendations(userId: string): Promise<Product[]> {
  try {
    return await recommendationService.forUser(userId);
  } catch {
    // Fallback: static bestsellers rather than a 500
    return bestsellerCache.getTop(10);
  }
}
```

Fallback strategies ranked by freshness:
- **Stale cache** — serve last-known-good from Redis even if TTL-expired (beats a 500)
- **Default/static response** — empty list, "not available right now" state
- **Degraded mode** — hide the feature entirely rather than showing an error

### Shed Load (Rate Limiting at Entry)

At the service entry point, **reject excess requests early** rather than queuing them. Shedding load under surge keeps latency predictable for accepted requests and prevents the queue from growing unboundedly, which would eventually degrade everyone.

```typescript
// express-rate-limit: return 429 immediately instead of queueing
const limiter = rateLimit({
  windowMs: 60_000,
  max: 100,               // 100 req/min per IP
  standardHeaders: true,
});
app.use('/api/', limiter);
```

At the infrastructure level, ALBs and API Gateways enforce rate limits before traffic reaches your service. Application-level Redis sliding-window rate limiting is common for per-user or per-tenant limits.

### The full resilience stack

The five patterns compose into a **defence-in-depth** strategy. Each addresses a different failure mode:

```
Incoming request
  → Shed Load       (reject if over capacity — protect the whole service)
  → Bulkhead        (reject if > max concurrent for this dependency — protect resource pools)
  → Timeout         (abort if no response within deadline — prevent hanging connections)
  → Circuit Breaker (skip if dependency is known broken — prevent pointless retries)
  → Fallback        (return degraded response — protect the user experience)
```

No single pattern is sufficient alone. Circuit Breaker without Timeout means you still hang waiting for the breaker to trip. Timeout without Fallback still returns an error to the user. The full stack applied to every external call is the minimum viable resilience posture for production SaaS.

**Interview framing:** "How do you protect a service from a slow downstream dependency?" — timeout + circuit breaker stops you holding connections open; bulkhead prevents resource starvation; fallback gives users a degraded but functional experience; shed load at the edge keeps latency predictable under surge. Mention that they compose — none is effective in isolation.

## Design Principles Priority Summary

| Topic                 | Priority      |
| --------------------- | ------------- |
| **SOLID**             |               |
| Single Responsibility | **Deep**      |
| Open/Closed           | **Deep**      |
| Dependency Inversion  | **Deep**      |
| **Design Patterns**   |               |
| Singleton             | **Refresh**   |
| Factory               | **Important** |
| Strategy              | **Important** |
| Repository            | **Critical**  |
| Circuit Breaker       | **Critical**  |
| Saga                  | **Critical**  |
| **Resilience Patterns** |             |
| Bulkhead              | **Important** |
| Timeout               | **Critical**  |
| Fallback              | **Important** |
| Shed load / rate limiting | **Important** |
| Full resilience stack | **Deep**      |

---


# 21. Messaging and Event Streaming

Asynchronous messaging is how services communicate without being tightly coupled in time. Instead of service A calling service B directly and waiting (synchronous, and broken if B is down), A writes a message to a broker and moves on; B processes it whenever it's ready. This buys you **decoupling** (producers and consumers don't know about each other), **load leveling** (a queue absorbs traffic spikes so a slow consumer isn't overwhelmed), and **resilience** (messages survive a consumer outage). The cost is complexity: you now reason about ordering, duplicates, failures, and eventual consistency. The single most important distinction a senior engineer must articulate is **message queue vs event streaming** — they look similar but have fundamentally different semantics.

## 21.0 Message Queue vs Event Streaming

This is the conceptual fork that determines which technology you reach for, and interviewers love it because candidates routinely conflate the two.

**Message Queue (RabbitMQ, ActiveMQ, SQS):** A message is a *command* or *task* aimed at being processed once. The broker tracks per-message delivery state. When a consumer acknowledges a message, **the broker deletes it** — it's gone. Think of it as a to-do list that gets crossed off. The model is destructive-read: once consumed and acked, no one can read that message again. Multiple consumers on one queue **compete** for messages (each message goes to exactly one of them), which is how you load-balance work across workers.

**Event Streaming (Kafka, Kinesis):** A message is an *event* — an immutable fact about something that happened ("OrderPlaced"). Events are appended to a **durable, ordered log** and **retained** for a configured period (hours, days, or forever), regardless of who has read them. Consumers track their *own* position (offset) in the log; reading doesn't delete anything. This makes two things possible that queues can't do: **replay** (a new consumer, or a fixed bug, can re-read the entire history from the beginning) and **multiple independent consumers** (the analytics pipeline, the email service, and the audit log each read the same events at their own pace from their own offset).

| Aspect | Message Queue | Event Streaming |
|---|---|---|
| **Unit** | Command/task (do this) | Event (this happened) |
| **After consumption** | Deleted (destructive read) | Retained for the whole retention window |
| **Replay history** | No | Yes (rewind the offset) |
| **Consumers** | Compete for messages (one wins) | Each consumer group reads everything independently |
| **Ordering** | Per queue (often weak) | Strong per partition |
| **Mental model** | To-do list | Append-only ledger / source of truth |
| **Typical use** | Background jobs, RPC, task distribution | Event sourcing, analytics, CDC, audit, fan-out to many systems |

A useful rule of thumb: if you're asking *"who should do this work?"* you want a queue; if you're stating *"this thing happened, and many parts of the system may care, now or later"* you want a stream.

## 21.1 Message Broker Patterns

A message broker decouples producers from consumers: the producer publishes a message and moves on; the broker holds it until a consumer is ready. The two fundamental patterns are **point-to-point** (one message, one consumer — used for work queues and task distribution) and **publish/subscribe** (one message, many consumers — used for event fans and notifications). Brokers differ most on durability guarantees, ordering, and replay: RabbitMQ offers per-queue acknowledgement with dead-letter queues for error handling; Kafka persists an ordered, replayable log that multiple independent consumer groups can read at different offsets; SQS provides managed queuing with at-least-once delivery and optional FIFO ordering. *The interview question is almost always:* "When would you choose Kafka over SQS?" — the answer hinges on replay, ordering, throughput, and consumer-group fan-out rather than just "Kafka is faster."

### Point-to-Point (Queue)

One producer, one consumer — each message is delivered to exactly one consumer. Multiple consumers share the load but each message is processed once. Used for task distribution and work queues.

```
Producer → Queue → Consumer
```

- One message, one consumer
- Load balancing across consumers

### Publish-Subscribe (Topic)

One producer, many subscribers — each subscriber gets an independent copy of each message. Used for event fanout, notifications, and triggering multiple downstream systems from a single event.

```
Producer → Topic → Subscriber 1
                 → Subscriber 2
                 → Subscriber 3
```

- One message, many subscribers
- Each subscriber gets a copy

## 21.2 RabbitMQ

RabbitMQ is the canonical traditional message broker. It implements **AMQP** and is built around a routing layer: producers publish to an **exchange**, not directly to a queue, and the exchange decides which queue(s) get the message based on **bindings** and the message's **routing key**. This indirection is RabbitMQ's superpower — the same publish can fan out, route by topic pattern, or target one queue, just by changing the exchange type. It excels at flexible routing, per-message acknowledgements, priorities, and TTLs, making it the default choice for task queues and RPC-style workloads where you need fine-grained control over delivery. It is *not* built for replay or long-term retention — once a message is acked, it's gone.

### Exchanges

Exchanges are RabbitMQ's routing layer — producers always publish to an exchange, never directly to a queue. The exchange type and message routing key determine which queues receive the message.

| Type        | Routing                                            |
| ----------- | -------------------------------------------------- |
| **Direct**  | Exact routing key match                            |
| **Topic**   | Pattern matching (logs.\* → logs.error, logs.info) |
| **Fanout**  | Broadcast to all queues                            |
| **Headers** | Match header attributes                            |

### Example

This wires a topic exchange to an error-log queue via a binding. Messages published with routing key `logs.error` are routed to `error-logs`; other `logs.*` keys are not.

```typescript
import amqp from "amqplib";

const connection = await amqp.connect("amqp://localhost");
const channel = await connection.createChannel();

// Declare exchange and queue
await channel.assertExchange("logs", "topic", { durable: true });
await channel.assertQueue("error-logs", { durable: true });
await channel.bindQueue("error-logs", "logs", "logs.error");

// Publish
channel.publish("logs", "logs.error", Buffer.from("Error occurred"));

// Consume
channel.consume("error-logs", (msg) => {
  if (msg) {
    console.log("Received:", msg.content.toString());
    channel.ack(msg); // Acknowledge
  }
});
```

### Dead Letter Queue (DLQ)

A **dead letter queue** is a side queue where messages go when they can't be processed successfully. Without one, a "poison message" — one that always fails (malformed payload, a bug, a referenced record that no longer exists) — gets retried forever, blocking the queue and burning resources. A DLQ breaks that loop: after a configured number of failed delivery attempts, the broker moves the message aside so the main queue keeps flowing, and you can inspect, fix, and **redrive** the failed messages later. This is a near-universal pattern: RabbitMQ implements it via a dead-letter exchange, and **SQS** lets you attach a DLQ with a `maxReceiveCount` redrive policy (after N receives without a successful delete, the message moves to the DLQ). The operational discipline that matters: **alert on DLQ depth** — a growing DLQ is one of the clearest signals that something downstream is broken.

```typescript
await channel.assertQueue("main-queue", {
  durable: true,
  deadLetterExchange: "dlx",
  deadLetterRoutingKey: "failed-messages",
});

await channel.assertExchange("dlx", "direct");
await channel.assertQueue("dlq", { durable: true });
await channel.bindQueue("dlq", "dlx", "failed-messages");

// Message goes to DLQ if:
// - Rejected with requeue=false
// - TTL expires
// - Queue length limit exceeded
```

## 21.3 Apache Kafka

Kafka is the dominant event-streaming platform — a distributed, append-only commit log built for massive throughput and durability. The core unit is the **partition**: an ordered, immutable sequence of records. A topic is split into partitions across brokers, which is how Kafka scales horizontally and parallelizes consumption. The key insight that trips people up: **ordering is only guaranteed within a partition, not across a topic.** The partition a record lands in is chosen by its key (`hash(key) % partitions`), so all records with the same key (e.g. the same `userId`) stay ordered relative to each other. Consumers belong to a **consumer group**; Kafka assigns each partition to exactly one consumer in the group, so adding consumers (up to the partition count) scales throughput. Because consumers track their own **offset** and the log is retained, Kafka supports replay and many independent consumer groups reading the same data — the things a queue cannot do.

### Concepts

Kafka's data model: a Topic is a logical stream split into ordered Partitions distributed across Brokers. Consumer Groups parallelise consumption — each partition is assigned to exactly one consumer in the group.

- **Topic**: Stream of messages
- **Partition**: Ordered log (messages append-only)
- **Consumer Group**: Multiple consumers share partitions
- **Offset**: Position in partition

### Producer

Producers publish records to a topic. The record's key determines which partition it lands in — records with the same key always go to the same partition, preserving ordering per key.

```typescript
import { Kafka } from "kafkajs";

const kafka = new Kafka({
  clientId: "my-app",
  brokers: ["localhost:9092"],
});

const producer = kafka.producer();
await producer.connect();

await producer.send({
  topic: "user-events",
  messages: [
    {
      key: "user-123", // Determines partition
      value: JSON.stringify({ event: "signup", userId: 123 }),
    },
  ],
});

await producer.disconnect();
```

### Consumer

Consumers track their position with an offset. `fromBeginning: true` replays the full partition log from the start — useful for reprocessing events after a bug fix or deploying a new downstream service.

```typescript
const consumer = kafka.consumer({ groupId: "analytics-group" });
await consumer.connect();
await consumer.subscribe({ topic: "user-events", fromBeginning: true });

await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    console.log({
      key: message.key.toString(),
      value: message.value.toString(),
      offset: message.offset,
    });
  },
});
```

### Partitions & Consumer Groups

A topic is split into N partitions; Kafka assigns one partition per consumer in a group. Adding consumers up to the partition count scales throughput linearly — beyond that, extra consumers sit idle.

```
Topic: user-events (3 partitions)

Producer:
  - key='user-1' → Partition 0
  - key='user-2' → Partition 1
  - key='user-3' → Partition 2

Consumer Group (3 consumers):
  - Consumer A → Partition 0
  - Consumer B → Partition 1
  - Consumer C → Partition 2

Messages with same key always go to same partition (ordering guaranteed per key)
```

### Kafka vs RabbitMQ

Choose Kafka for high-throughput event streaming and replay; RabbitMQ for flexible routing, per-message acknowledgements, and task queues.

|                 | Kafka                           | RabbitMQ                     |
| --------------- | ------------------------------- | ---------------------------- |
| **Model**       | Event streaming (log)           | Message broker (queue)       |
| **Persistence** | Always (append-only log)        | Optional                     |
| **Replay**      | Yes (rewind offset)             | No (consumed = deleted)      |
| **Throughput**  | Very high (millions/sec)        | High (tens of thousands/sec) |
| **Ordering**    | Per partition                   | Per queue                    |
| **Use case**    | Event sourcing, analytics, logs | Task queues, RPC             |

## 21.4 AWS SQS / SNS

SQS and SNS are AWS's fully managed, serverless messaging primitives — no brokers to run, they scale automatically, and you pay per request. They are deliberately simple and pair together. **SQS** is a queue (point-to-point, competing consumers, destructive read). **SNS** is pub/sub (one publish, fan-out to many subscribers). The canonical AWS pattern is **SNS → SQS fan-out**: publish an event once to an SNS topic, and have several SQS queues subscribe, so each downstream service gets its own durable copy to process at its own pace. SQS comes in two flavors: **Standard** (near-unlimited throughput, at-least-once delivery, best-effort ordering) and **FIFO** (strict ordering and exactly-once processing within a message group, at lower throughput).

### SQS (Queue)

```typescript
import {
  SQSClient,
  SendMessageCommand,
  ReceiveMessageCommand,
} from "@aws-sdk/client-sqs";

const client = new SQSClient({ region: "us-east-1" });

// Send
await client.send(
  new SendMessageCommand({
    QueueUrl: "https://sqs.us-east-1.amazonaws.com/123/my-queue",
    MessageBody: JSON.stringify({ event: "user-signup", userId: 123 }),
    DelaySeconds: 10, // Delay delivery
  }),
);

// Receive
const response = await client.send(
  new ReceiveMessageCommand({
    QueueUrl: "https://sqs.us-east-1.amazonaws.com/123/my-queue",
    MaxNumberOfMessages: 10,
    WaitTimeSeconds: 20, // Long polling
  }),
);

for (const message of response.Messages || []) {
  console.log(message.Body);

  // Delete after processing
  await client.send(
    new DeleteMessageCommand({
      QueueUrl: "...",
      ReceiptHandle: message.ReceiptHandle,
    }),
  );
}
```

### SQS FIFO

```typescript
await client.send(
  new SendMessageCommand({
    QueueUrl: "https://sqs.us-east-1.amazonaws.com/123/my-queue.fifo",
    MessageBody: JSON.stringify({ event: "order-created", orderId: 1 }),
    MessageGroupId: "order-123", // Messages in same group are ordered
    MessageDeduplicationId: "unique-id", // Prevent duplicates
  }),
);
```

### SNS (Topic / Pub-Sub)

```typescript
import { SNSClient, PublishCommand } from "@aws-sdk/client-sns";

const client = new SNSClient({ region: "us-east-1" });

await client.send(
  new PublishCommand({
    TopicArn: "arn:aws:sns:us-east-1:123:user-events",
    Message: JSON.stringify({ event: "user-signup", userId: 123 }),
  }),
);

// Subscribers: SQS, Lambda, HTTP, Email, SMS
```

### SNS + SQS Fanout Pattern

```
SNS Topic (user-events)
  → SQS Queue (email-service)
  → SQS Queue (analytics-service)
  → Lambda (notification-service)

Each subscriber gets a copy of the message
```

## 21.5 Amazon Kinesis

Kinesis is AWS's managed event-streaming service — conceptually Kafka-as-a-service for the AWS ecosystem. Like Kafka it's a durable, replayable, ordered log, but the unit of scaling is the **shard** rather than the partition. Each shard provides fixed capacity (1 MB/s or 1,000 records/s in, 2 MB/s out), and you scale by splitting/merging shards. The partition key you supply hashes a record to a shard, and ordering is guaranteed within a shard. Records are retained 24 hours by default (extendable to 365 days), so multiple consumers can read independently and replay.

There are several Kinesis products, and the distinction matters in interviews:
- **Kinesis Data Streams** — the raw, low-latency streaming log you read/write with code or Lambda. You manage consumers and offsets. This is the Kafka analogue.
- **Kinesis Data Firehose** — a zero-code "load-and-forget" pipe that batches streaming data and delivers it to S3, Redshift, or OpenSearch, with optional transform via Lambda. Use it for ingestion into a data lake/warehouse, not for application consumers.
- **Kinesis Data Analytics** — run SQL or Apache Flink over the stream for real-time aggregations.

```typescript
import { KinesisClient, PutRecordCommand } from "@aws-sdk/client-kinesis";

const client = new KinesisClient({ region: "us-east-1" });

await client.send(new PutRecordCommand({
  StreamName: "clickstream",
  PartitionKey: "user-123",          // ← hashed to a shard; same key → same shard → ordered
  Data: Buffer.from(JSON.stringify({ event: "page_view", url: "/home" })),
}));
// Consumers (often Lambda via an event-source mapping) read per-shard in order,
// checkpoint their position, and can replay within the retention window.
```

**Kinesis vs Kafka:** Kinesis is fully managed (no clusters to operate) and integrates natively with Lambda/Firehose/IAM, but it's AWS-locked, has hard per-shard limits, and a max 365-day retention. Kafka is more flexible, higher-ceiling, has a richer ecosystem (Kafka Connect, Streams, infinite tiered storage), but you (or MSK) must operate it. Choose Kinesis for AWS-native, moderate-scale streaming where ops simplicity wins; choose Kafka/MSK for very high throughput, long retention, or multi-cloud.

## 21.6 ActiveMQ / Amazon MQ and EventBridge

**ActiveMQ** is a mature, traditional broker from the Java/enterprise world. Its defining feature is **JMS** (Java Message Service) support plus a wide range of protocols (AMQP, MQTT, STOMP, OpenWire). You reach for ActiveMQ — usually via **Amazon MQ**, the managed version that also offers managed RabbitMQ — when you're **migrating an existing JMS/enterprise application to the cloud** and need protocol compatibility without rewriting messaging code. For brand-new cloud-native systems you'd typically pick SQS/SNS, Kafka, or Kinesis instead; Amazon MQ exists primarily as a lift-and-shift path. Functionally it's a queue/topic broker like RabbitMQ — destructive reads, no Kafka-style replay.

**Amazon EventBridge** is worth knowing as the modern AWS event bus. It's pub/sub like SNS but adds **content-based routing** (rules that match on the event JSON), a **schema registry**, and dozens of **SaaS and AWS service integrations** as event sources/targets. The rough guidance: use **SNS** for simple, high-throughput fan-out (especially SNS→SQS); use **EventBridge** when you need rich routing rules, event filtering, or integration with many AWS services and third-party SaaS in an event-driven architecture.

## 21.7 Choosing a Technology — Side by Side

| | Kafka | Kinesis | RabbitMQ | ActiveMQ (Amazon MQ) | SQS | SNS | EventBridge |
|---|---|---|---|---|---|---|---|
| **Category** | Streaming log | Streaming log | Queue broker | Queue broker (JMS) | Queue | Pub/sub | Event bus (pub/sub) |
| **Managed?** | Self/MSK | Yes (AWS) | Self/Amazon MQ | Yes (Amazon MQ) | Yes | Yes | Yes |
| **Replay** | Yes | Yes (retention) | No | No | No | No | No |
| **Ordering** | Per partition | Per shard | Per queue (weak) | Per queue | FIFO only | No | No |
| **Throughput** | Very high | High (per-shard caps) | High | Moderate | Very high | Very high | Moderate |
| **Multi-consumer** | Many groups, independent | Many, independent | Compete (or fanout exchange) | Compete / topics | Compete | Fan-out copies | Fan-out + rules |
| **Best for** | Event sourcing, analytics, CDC | AWS-native streaming/ingest | Task queues, RPC, routing | JMS lift-and-shift | Background jobs | Simple fan-out | Routed event-driven AWS apps |

## 21.8 Delivery Guarantees

Every messaging system makes a promise about how often a message is delivered, and understanding the three levels — and why true exactly-once is so hard — is core senior knowledge. The tension is fundamental: to guarantee a message is *never lost*, the system must retry until it gets an acknowledgement; but if the original was actually processed and only the ack was lost, the retry creates a *duplicate*. You can't escape this with the network alone. In practice almost all systems give you **at-least-once** and you achieve *effective* exactly-once at the application layer by making consumers **idempotent** (so a duplicate is a no-op). "Exactly-once" offered by brokers (e.g. Kafka transactions) holds within that broker's boundary but doesn't extend to your external side effects — which is why idempotency keys and the transactional outbox below are the real-world tools.

| Guarantee         | Behavior                                                   |
| ----------------- | ---------------------------------------------------------- |
| **At-most-once**  | May lose messages (fire and forget)                        |
| **At-least-once** | May deliver duplicates (retry on failure)                  |
| **Exactly-once**  | No loss, no duplicates (hardest, often "effectively once") |

### Achieving Exactly-Once

```typescript
// 1. Idempotent processing
async function processMessage(message: Message) {
  const processed = await db.processedMessages.exists(message.id);
  if (processed) {
    return; // Already
     processed, skip
  }

  await doWork(message);
  await db.processedMessages.save(message.id);
}

// 2. Transactional outbox pattern
async function createOrder(data: CreateOrderDto) {
  await db.transaction(async (tx) => {
    // Insert order
    const order = await tx.orders.create(data);

    // Insert outbox message
    await tx.outbox.create({
      aggregateId: order.id,
      eventType: "OrderCreated",
      payload: order,
    });
  });

  // Background worker publishes outbox messages to message broker
}
```

## Messaging Priority Summary

| Topic                               | Priority      |
| ----------------------------------- | ------------- |
| **Concepts**                        |               |
| Queue vs event streaming            | **Critical**  |
| Point-to-point vs pub/sub           | **Critical**  |
| **RabbitMQ**                        |               |
| Exchanges (direct, topic, fanout)   | **Important** |
| Dead Letter Queues + redrive        | **Deep**      |
| Acknowledgments                     | **Deep**      |
| **Apache Kafka**                    |               |
| Topics, partitions, consumer groups | **Critical**  |
| Ordering guarantees                 | **Critical**  |
| Kafka vs RabbitMQ                   | **Deep**      |
| **AWS messaging**                   |               |
| SQS (standard vs FIFO)              | **Important** |
| SNS (pub-sub) + SNS→SQS fan-out     | **Important** |
| Kinesis (Streams/Firehose) + vs Kafka | **Important** |
| EventBridge (routing) vs SNS        | **Learn**     |
| ActiveMQ / Amazon MQ (JMS)          | **Know**      |
| Technology comparison table         | **Deep**      |
| **Delivery Guarantees**             |               |
| At-most/least/exactly-once          | **Critical**  |
| Idempotent processing               | **Critical**  |
| Transactional outbox                | **Important** |

---


# 22. Data Structures & Algorithms

For a 10-year engineer, the DSA interview is rarely about exotic algorithms — it's about recognizing which of a handful of patterns a problem maps to, and implementing it cleanly while reasoning about time/space complexity out loud. Most coding-round problems reduce to: two pointers, sliding window, hashing for O(1) lookup, tree/graph traversal (DFS/BFS), or dynamic programming. The senior differentiator is communication: state the brute-force approach, give its complexity, then optimize and justify the trade-off. The patterns below are the highest-leverage ones to drill.

## 22.1 Essential Patterns

### Two Pointers

Use two indices moving through a structure (from both ends, or at different speeds) to turn an O(n²) nested loop into a single O(n) pass. Classic for sorted-array problems, palindromes, and in-place partitioning — it works because moving a pointer deterministically eliminates possibilities without re-checking them.

```typescript
// Palindrome
function isPalindrome(s: string): boolean {
  let left = 0,
    right = s.length - 1;
  while (left < right) {
    if (s[left] !== s[right]) return false;
    left++;
    right--;
  }
  return true;
}

// Two Sum (sorted array)
function twoSum(nums: number[], target: number): number[] {
  let left = 0,
    right = nums.length - 1;
  while (left < right) {
    const sum = nums[left] + nums[right];
    if (sum === target) return [left, right];
    if (sum < target) left++;
    else right--;
  }
  return [];
}
```

### Sliding Window

A specialization of two pointers for contiguous subarrays/substrings: maintain a window `[start, end]`, expand `end` to include elements and shrink `start` when a constraint is violated, updating an answer as you go. It converts "examine every subarray" (O(n²) or worse) into O(n) because each element enters and leaves the window at most once. Reach for it on "longest/shortest/max-sum substring or subarray with condition X" problems.

```typescript
// Max sum subarray of size k
function maxSubarraySum(arr: number[], k: number): number {
  let maxSum = 0,
    windowSum = 0;

  for (let i = 0; i < k; i++) {
    windowSum += arr[i];
  }
  maxSum = windowSum;

  for (let i = k; i < arr.length; i++) {
    windowSum += arr[i] - arr[i - k];
    maxSum = Math.max(maxSum, windowSum);
  }

  return maxSum;
}

// Longest substring without repeating
function lengthOfLongestSubstring(s: string): number {
  const seen = new Set<string>();
  let left = 0,
    maxLen = 0;

  for (let right = 0; right < s.length; right++) {
    while (seen.has(s[right])) {
      seen.delete(s[left]);
      left++;
    }
    seen.add(s[right]);
    maxLen = Math.max(maxLen, right - left + 1);
  }

  return maxLen;
}
```

### Trees (DFS/BFS)

Tree and graph problems split along one axis: do you want to go deep or go wide? **DFS** (recursion or an explicit stack) follows one branch to its end before backtracking — natural for path-finding, subtree aggregation, and the three traversal orders (pre/in/post). **BFS** (a queue) explores level by level — the right choice when you need shortest path in an unweighted graph or "process nodes by distance from the root." Knowing which to reach for, and stating why, is the senior signal here. The level-order BFS below uses the classic "snapshot the queue length" trick to process exactly one level per outer iteration.

```typescript
class TreeNode {
  val: number;
  left: TreeNode | null;
  right: TreeNode | null;
}

// DFS Inorder
function inorder(root: TreeNode | null): number[] {
  const result: number[] = [];
  function dfs(node: TreeNode | null) {
    if (!node) return;
    dfs(node.left);
    result.push(node.val);
    dfs(node.right);
  }
  dfs(root);
  return result;
}

// BFS Level Order
function levelOrder(root: TreeNode | null): number[][] {
  if (!root) return [];
  const result: number[][] = [];
  const queue: TreeNode[] = [root];

  while (queue.length) {
    const level: number[] = [];
    const size = queue.length;

    for (let i = 0; i < size; i++) {
      const node = queue.shift()!;
      level.push(node.val);
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }
    result.push(level);
  }
  return result;
}
```

### Dynamic Programming

DP applies when a problem has **optimal substructure** (the answer is built from answers to smaller subproblems) and **overlapping subproblems** (those smaller pieces recur). The move is to define a state, write the recurrence relating it to smaller states, then either memoize top-down or fill a table bottom-up to avoid recomputing. In interviews, the hard part is rarely the code — it's articulating *what `dp[i]` means* and *why the recurrence is correct*. Coin Change below: `dp[i]` is the fewest coins to make amount `i`, and each coin offers a candidate transition `dp[i - coin] + 1`.

```typescript
// Coin Change
function coinChange(coins: number[], amount: number): number {
  const dp = new Array(amount + 1).fill(Infinity);
  dp[0] = 0;

  for (let i = 1; i <= amount; i++) {
    for (const coin of coins) {
      if (i >= coin) {
        dp[i] = Math.min(dp[i], dp[i - coin] + 1);
      }
    }
  }

  return dp[amount] === Infinity ? -1 : dp[amount];
}
```

## DSA Priority Summary

| Topic                        | Priority    |
| ---------------------------- | ----------- |
| Arrays, Strings, Hash Maps   | **Refresh** |
| Two Pointers, Sliding Window | **Learn**   |
| Trees (DFS, BFS)             | **Learn**   |
| Binary Search                | **Refresh** |
| Dynamic Programming (basics) | **Know**    |

**Practice:** 20-30 LeetCode Medium problems

---


# 23. Real-Time Systems

Plain HTTP request/response can't push: the client must ask before the server can answer. Real-time systems break that constraint with a persistent connection. The two workhorses are **WebSockets** (a full-duplex TCP channel after an HTTP upgrade handshake — either side can send any time) and **Server-Sent Events** (a long-lived HTTP response that streams server→client only). The interview question is almost always "which would you pick?" — and the honest answer is that SSE is simpler and auto-reconnects for one-way feeds (notifications, live dashboards, token streaming), while WebSockets are warranted only when you genuinely need low-latency client→server traffic too (chat, multiplayer, collaborative editing). Polling/long-polling remains the fallback where neither is available.

## 23.1 WebSockets

WebSockets upgrade an HTTP connection to a persistent, full-duplex TCP channel — both client and server can send frames at any time without the overhead of re-establishing a connection. The handshake is HTTP, so WebSockets traverse the same ports (80/443) and proxies that regular HTTP does, but the protocol switches to `ws://` or `wss://`. The critical production concern is **horizontal scaling**: because a WebSocket connection is stateful and pinned to a single server process, you cannot load-balance freely without a shared pub/sub layer (Redis pub/sub or a dedicated broker) that fans messages out to all server instances holding connections for a given user or room.

### Server

The `ws` library provides a minimal WebSocket server. Broadcasting requires iterating `wss.clients` — in production with multiple server instances, add a Redis pub/sub layer to fan out to all connections.

```typescript
import { WebSocketServer } from "ws";

const wss = new WebSocketServer({ port: 8080 });

wss.on("connection", (ws) => {
  ws.on("message", (data) => {
    // Broadcast to all
    wss.clients.forEach((client) => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(data);
      }
    });
  });
});
```

### Client

The browser's native WebSocket API connects to a `wss://` endpoint. In production React apps, use a library like `socket.io-client` for automatic reconnection and room-based message routing.

```typescript
const ws = new WebSocket("wss://example.com");

ws.onmessage = (event) => {
  console.log("Message:", JSON.parse(event.data));
};

ws.send(JSON.stringify({ type: "chat", message: "Hello" }));
```

## 23.2 Server-Sent Events

Server-Sent Events (SSE) are a simpler, HTTP-native alternative when you only need server-to-client push. The server holds an open `text/event-stream` response and writes newline-delimited `data:` frames; the browser's built-in `EventSource` API handles reconnection automatically. Because SSE runs over plain HTTP/2, it benefits from multiplexing and works without special proxy configuration — a meaningful operational advantage over WebSockets in environments where load balancers or CDNs complicate WebSocket upgrades. SSE is the right default for notification feeds, live dashboards, and progress streams; reach for WebSockets only when the client also needs to send data or when latency and message volume demand true bidirectional framing.

```typescript
// Server
app.get("/events", (req, res) => {
  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");

  setInterval(() => {
    res.write(`data: ${JSON.stringify({ time: Date.now() })}\n\n`);
  }, 5000);
});

// Client
const eventSource = new EventSource("/events");
eventSource.onmessage = (event) => {
  console.log(JSON.parse(event.data));
};
```

## 23.3 WebSockets vs SSE

The choice comes down to directionality and operational simplicity. SSE is unidirectional (server → client), runs over standard HTTP, auto-reconnects, and requires no special infrastructure — ideal for feeds, dashboards, and progress updates. WebSockets are bidirectional, require a persistent TCP upgrade, and demand a shared pub/sub layer when scaled horizontally. A common mistake is defaulting to WebSockets "because it's real-time" when SSE would suffice and is far easier to deploy, monitor, and cache-proxy. Use WebSockets when the client must send messages at high frequency or low latency (chat, collaborative editing, multiplayer games).

|           | WebSocket     | SSE                    |
| --------- | ------------- | ---------------------- |
| Direction | Bidirectional | Server → Client        |
| Protocol  | TCP           | HTTP                   |
| Use case  | Chat, games   | Notifications, updates |

## Real-Time Priority

| Topic              | Priority      |
| ------------------ | ------------- |
| WebSockets         | **Important** |
| Server-Sent Events | **Learn**     |
| Presence systems   | **Learn**     |

---

_End of Part 3. Continue to **Part 4** (DevOps & CI/CD) in [`04_DevOps_24.md`](./04_DevOps_24.md), or return to the [README](./README.md)._
