# SENIOR FULL-STACK ENGINEER — COMPLETE KNOWLEDGE MAP

## PART 9: Section 37

### Generative AI & Prompt Engineering

---

# 37. GENERATIVE AI & PROMPT ENGINEERING

LLMs are now infrastructure. Senior engineers are expected to understand them well enough to integrate them safely, prompt them effectively, and ship LLM-powered features to production — not just call an API and hope. Whether you're wiring up a RAG pipeline, building a tool-calling agent, or deciding when to use a model at all, the concepts here are increasingly table-stakes in interviews and day-to-day work. The section moves from *how LLMs work* (fundamentals), through *how to use them well* (prompting, RAG, tool use), and lands on *what production concerns matter* (evals, cost, safety). Cross-references: §37.4 uses the Anthropic SDK patterns from §1.3 (async); production observability ties to §26.

## 37.1 LLM Fundamentals

LLMs are **token predictors** — they take a sequence of tokens and predict, over and over, what the next token should be. Everything the model produces (code, reasoning, answers) is this one operation, tuned by training to be useful. Understanding tokens, context windows, and sampling parameters lets you reason about cost, latency, and why models sometimes produce unexpected output.

### Tokens and Context Windows

A token is a chunk of text produced by the model's tokenizer — roughly 0.75 words per token on average, though it varies (common words are single tokens; rare words and non-English text are often several). The **context window** is the model's working memory: everything the model "knows" during generation — your system prompt, conversation history, retrieved documents, the user's question — must fit inside it.

| Model | Context window | Notes |
| --- | --- | --- |
| GPT-4o | 128K tokens | ~96K words |
| Claude 3.5 Sonnet | 200K tokens | ~150K words |
| Llama 3.1 70B | 128K tokens | Open-weight, self-hostable |
| Gemini 1.5 Pro | 1M tokens | Large document / video use cases |

**Cost and latency scale with token count.** Input tokens (your prompt) and output tokens (the model's response) are billed separately; output is 3–5× more expensive per token. For high-volume applications, prompt length is the primary cost lever.

### Temperature and Sampling

Sampling parameters control *how* the model picks the next token from its probability distribution.

| Parameter | Effect | Typical values |
| --- | --- | --- |
| **temperature** | Scales probabilities — 0 = always pick highest-probability token (deterministic), >1 = more random | 0 for code/facts, 0.7 for creative/varied |
| **top-p** (nucleus) | Only sample from tokens whose cumulative probability ≤ p | 0.9–0.95 typical |
| **top-k** | Only consider the k highest-probability tokens | Less common than top-p |
| **max_tokens** | Hard cap on output length | Set explicitly to control cost |
| **stop sequences** | Halt generation when a specified string is encountered | Enforce output delimiters; prevent runaway generation |
| **frequency_penalty** | Penalise tokens proportional to how often they've already appeared | Reduces repetition in long outputs |
| **presence_penalty** | Penalise tokens that have appeared at all (binary) | Encourages topical diversity |

Temperature 0 gives you reproducible output — important for testing and code generation. Higher temperature is useful when you want variety (brainstorming, paraphrasing) but makes responses less predictable.

### Model Families

```
Commercial (API only):
  OpenAI:    GPT-4o (fast, multimodal), o1/o3 (reasoning/CoT models, slower + expensive)
  Anthropic: Claude 3.5 Sonnet (best coding + reasoning), Claude 3 Haiku (fast + cheap)
  Google:    Gemini 1.5/2 Pro (long context), Flash (fast + cheap)

Open-weight (self-hostable / local):
  Meta:      Llama 3.1 8B/70B/405B
  Mistral:   Mistral 7B, Mixtral 8×7B (MoE)
  Microsoft: Phi-3 (small, efficient)
```

**Capability vs cost trade-off:** frontier models (GPT-4o, Claude 3.5 Sonnet) have the highest capability but cost 10–100× more per token than smaller models. For production, the default strategy is: use the cheapest model that passes your evals (§37.5).

### Limitations to Internalize

- **Hallucination** — models generate plausible-sounding text even when they don't know the answer. They have no way to distinguish "I know this" from "I'm guessing confidently."
- **Training cutoff** — knowledge is frozen at the training cutoff; events after it don't exist to the model. RAG (§37.3) or web search tools are the remedy.
- **Lost in the middle** — model attention degrades for content in the middle of very long contexts; the beginning and end receive the most weight. Place critical instructions at the start and/or end.
- **Context is stateless** — every API call starts fresh; the model has no memory of previous conversations unless you re-send them. State management is your responsibility.
- **Biased outputs** — models reflect biases in training data and can produce unfair or harmful results for certain demographic groups. Test for demographic disparities; add explicit fairness instructions when the domain warrants it.
- **Jailbreaks and data poisoning** — adversarial prompts can bypass safety guardrails (jailbreaks); maliciously crafted training data can introduce persistent backdoors. Defenses overlap with prompt injection mitigations (§37.2).

## 37.2 Prompting Techniques

The quality of your output depends more on prompt construction than on model choice. The techniques below are ordered from simplest to most powerful; for most production use cases, zero-shot with good instructions, few-shot examples, and structured output format covers 90% of what you need.

### Prompt Structure

Well-structured prompts share a consistent anatomy. Not every component is needed every time — include what the task requires.

| Component | Purpose | Example |
| --- | --- | --- |
| **Role** | Tell the model who it is | "You are a senior TypeScript engineer." |
| **Context** | Background the model needs | "We use NestJS with PostgreSQL." |
| **Instruction** | The specific task | "Review this function for security issues." |
| **Data** | The content to act on | The actual code snippet |
| **Output format** | How results should be structured | "Return JSON: `{ issues: Issue[] }`" |
| **Examples** | Demonstration of expected behaviour | Few-shot examples (see below) |
| **Constraints** | What to avoid or limit | "Focus only on auth logic. Max 5 issues." |
| **Evaluation criteria** | What a good answer looks like | "Prioritise OWASP Top 10 categories." |

A minimal production prompt is **Role + Instruction + Output format**. Add Context and Data for grounding; add Examples when format or tone matters.

### Zero-Shot and Few-Shot

**Zero-shot:** ask the model to do the task with only instructions, no examples. Works well for tasks the model has seen extensively in training (summarization, translation, common transformations).

**Few-shot:** provide 2–5 labelled examples of (input → output) before the actual task. Examples anchor the model on format, tone, and edge-case handling — especially effective when zero-shot produces inconsistent structure.

```typescript
// Few-shot: extract severity from error messages
const prompt = `
Classify the severity of these error messages as "critical", "warning", or "info".

Error: "OutOfMemoryError: Java heap space" → critical
Error: "WARN: retry attempt 2/3 for endpoint /api/users" → warning
Error: "INFO: Cache warmed with 12,400 entries" → info
Error: "FATAL: database connection pool exhausted after 30s" → critical

Error: "${userMessage}" →`;
```

The arrow `→` and consistent format are the real signal — the model pattern-matches to complete the sequence.

### Chain-of-Thought (CoT)

CoT prompting asks the model to *reason step by step* before answering. For tasks requiring multi-step logic (math, code debugging, complex planning), CoT significantly improves accuracy because intermediate reasoning steps catch errors before the final answer.

```typescript
// Without CoT: model jumps to (often wrong) conclusion
"Is 17 × 24 larger than 400? Answer yes or no."

// With CoT: model reasons through it
"Is 17 × 24 larger than 400? Think step by step, then answer yes or no."
// → "17 × 20 = 340. 17 × 4 = 68. 340 + 68 = 408. 408 > 400. Yes."
```

**"Think step by step"** is the minimal CoT trigger. For complex tasks, an explicit scratchpad format improves reliability:

```typescript
const systemPrompt = `
Before answering, think through the problem in <thinking> tags.
Then provide your final answer in <answer> tags.

Example:
<thinking>
The user asks about X. First consider Y, then Z...
</thinking>
<answer>
Final response here.
</answer>
`;
```

### Tree-of-Thoughts (ToT)

Tree-of-Thoughts generalises CoT by exploring **multiple reasoning paths in parallel**, evaluating them, and selecting the best branch — rather than committing to a single chain. Useful for tasks with combinatorial solution spaces: algorithm design, complex planning, creative generation.

```
Problem: "Design a caching strategy for this API endpoint"

Branch A: Redis + TTL-based expiration
  pros: simple, battle-tested
  cons: stale data between updates

Branch B: Event-driven cache invalidation (pub/sub)
  pros: always consistent
  cons: more complex, extra latency

Branch C: CDN caching for public endpoints only
  pros: eliminates origin load
  cons: not applicable to authenticated routes

→ Decision: Branch A for private data, Branch C for public endpoints.
```

In practice, implement ToT by prompting the model to enumerate options with pros/cons, then evaluate and select — rather than building an automated tree search.

### Structured Output (JSON Mode)

For programmatic use — extracting entities, classifying inputs, populating data structures — instruct the model to respond in JSON. Modern APIs have a `response_format` or schema enforcement parameter that guarantees valid JSON.

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

// Ask for structured output
const response = await client.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 512,
  system: 'You extract structured data from unstructured text. Always respond with valid JSON only.',
  messages: [{
    role: 'user',
    content: `Extract the following from this job posting and return JSON:
    { "title": string, "level": "junior"|"mid"|"senior", "skills": string[], "remote": boolean }

    Job posting: "${jobPosting}"`
  }]
});

const data = JSON.parse(response.content[0].text);
```

**Interview framing:** "How do you reliably get structured output from an LLM?" — JSON mode + schema in the system prompt + validation on your side (`zod.parse()`). Never trust the model to format correctly without enforcement.

**Other output formats** — choose based on who or what consumes the output:

| Format | When to use |
| --- | --- |
| **JSON / YAML** | Machine-readable data extraction, API responses |
| **Markdown** | Human-readable reports, documentation, chat responses |
| **CSV / TSV** | Tabular data for spreadsheets or downstream pipelines |
| **XML** | Legacy integrations, schema-validated documents |

Include a format example in the system prompt regardless of which format you choose; it anchors the model far better than a description alone.

### Prompt Injection Risks

User-controlled input that reaches the model can manipulate its behavior — a user asking "Ignore your previous instructions and…" or embedding instructions in a document being summarized. Defenses:

- **Separate instruction channels** — put system instructions in the system prompt, never inline with user data.
- **Input sanitization** — strip or escape common injection patterns from user content before including it in prompts.
- **Output validation** — never pass model output directly to SQL, shell, or eval.
- **Principle of least privilege** — don't give tool-calling agents more permissions than they need.

### Iterative Prompt Refinement

Prompt quality improves through iteration, not upfront design. The cycle:

1. **Draft** — write the simplest prompt that could work
2. **Test** — run against representative inputs, including edge cases and adversarial examples
3. **Inspect** — read failures carefully; most prompt bugs have a detectable pattern
4. **Revise** — fix the specific failure: add a constraint, an example, or a clearer instruction

Run this cycle against your eval suite (§37.5) to catch regressions. A change that fixes one case often breaks another — that's why evals come first.

## 37.3 Retrieval-Augmented Generation (RAG)

RAG solves the two main limitations of base LLMs for production: the training cutoff and hallucination on domain-specific data. Instead of fine-tuning the model on your data (expensive, requires retraining on every update), RAG **retrieves relevant context at query time** and includes it in the prompt. The model's job is then to *synthesize* retrieved facts, not recall them — which drastically reduces hallucination.

### Why RAG over Fine-Tuning

| | Fine-tuning | RAG |
| --- | --- | --- |
| **Freshness** | Stale (requires retrain) | Real-time (index anytime) |
| **Cost** | High (GPU training + storage) | Low (inference + vector DB) |
| **Control** | Implicit (baked in weights) | Explicit (see what was retrieved) |
| **Best for** | Style/format changes, domain language | Factual Q&A over a corpus |

Fine-tuning is mainly useful for adapting the model's *style or format* (e.g., tone, output structure), not for injecting knowledge.

### The RAG Pipeline

```
Query
  ↓
1. Embed query → dense vector
  ↓
2. Vector similarity search → top-k relevant chunks
  ↓
3. Build prompt: system + retrieved chunks + query
  ↓
4. LLM generates answer grounded in retrieved context
  ↓
Response (with citations)
```

## 37.4 Tool Use & Agentic Patterns

Tool use lets the model call functions you define — search the web, query a database, execute code, send an email. This is what transforms a chat model into an **agent**: a system that reasons about what to do, calls tools to get information or take actions, observes the results, and repeats until the task is complete.

### Tool / Function Calling

You define tools as JSON schemas; the model decides when to call them and with what arguments; you execute the call and return the result.

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

const tools: Anthropic.Tool[] = [
  {
    name: 'get_weather',
    description: 'Get current weather for a city',
    input_schema: {
      type: 'object',
      properties: {
        city: { type: 'string', description: 'City name' },
        units: { type: 'string', enum: ['celsius', 'fahrenheit'], default: 'celsius' }
      },
      required: ['city']
    }
  },
  {
    name: 'search_docs',
    description: 'Search internal documentation',
    input_schema: {
      type: 'object',
      properties: {
        query: { type: 'string' }
      },
      required: ['query']
    }
  }
];

// Tool execution map
const toolHandlers: Record<string, (input: any) => Promise<string>> = {
  get_weather: async ({ city, units }) => {
    const data = await weatherAPI.get(city, units);
    return JSON.stringify(data);
  },
  search_docs: async ({ query }) => {
    const chunks = await retrieve(query);
    return chunks.join('\n\n');
  }
};

async function runAgentLoop(userMessage: string) {
  const messages: Anthropic.MessageParam[] = [
    { role: 'user', content: userMessage }
  ];

  while (true) {
    const response = await client.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 4096,
      tools,
      messages,
    });

    // Check if model wants to use a tool
    if (response.stop_reason === 'tool_use') {
      const toolUseBlocks = response.content.filter(
        (b): b is Anthropic.ToolUseBlock => b.type === 'tool_use'
      );

      // Execute tools and collect results
      const toolResults: Anthropic.ToolResultBlockParam[] = await Promise.all(
        toolUseBlocks.map(async (toolUse) => ({
          type: 'tool_result' as const,
          tool_use_id: toolUse.id,
          content: await toolHandlers[toolUse.name](toolUse.input),
        }))
      );

      // Add assistant response and tool results to history
      messages.push({ role: 'assistant', content: response.content });
      messages.push({ role: 'user', content: toolResults });

    } else {
      // Model is done — return the final text response
      const finalText = response.content.find(b => b.type === 'text');
      return finalText?.text ?? '';
    }
  }
}
```

### The ReAct Pattern

ReAct (**Re**asoning + **Act**ing) is the standard agentic loop: the model alternates between reasoning about the current state and taking an action (tool call), observing the result, then reasoning again.

```
User: "Find all PRs merged to main this week and summarize the changes"

Thought: I need to list recent PRs. I'll call the GitHub tool.
Action: search_github_prs({ repo: "org/app", merged_since: "2024-01-15" })
Observation: [PR #234: "Add rate limiting", PR #235: "Fix auth bug", ...]

Thought: I have the PR list. Now I need the diff for each.
Action: get_pr_diff({ pr_number: 234 })
Observation: "+++ rate limiter middleware using Redis token bucket..."

... (repeat for each PR) ...

Thought: I have all the diffs. I can now write the summary.
Answer: "This week 2 PRs were merged: (1) Rate limiting added via Redis token bucket middleware..."
```

The loop terminates when the model produces a final answer instead of a tool call. Preventing infinite loops: set a `max_iterations` guard; fail gracefully if exceeded.

### When Agents vs Single-Shot

| | Single-shot | Agent / multi-step |
| --- | --- | --- |
| **Use when** | Task is self-contained, all context available | Task requires external info, sequential decisions, or side effects |
| **Cost** | 1 LLM call | N LLM calls (N = steps) |
| **Latency** | Low | High (each step adds latency) |
| **Reliability** | High | Lower (errors compound; needs retries + guardrails) |

Default to single-shot. Add the agent loop only when the task genuinely requires iteration — you often discover during prototyping that you can pre-fetch the data and do it in one call.

### Guardrails and Safety

- **Input validation** — classify or filter user input before it reaches the agent; block prompt injection attempts.
- **Tool permission scoping** — agents should only have access to the tools the task requires (principle of least privilege).
- **Output validation** — parse and validate tool call arguments before executing; never `eval()` model-generated code.
- **Human-in-the-loop** — for irreversible actions (send email, delete record, charge card), pause and require human confirmation.
- **Max iterations** — always cap the agent loop to prevent runaway cost or infinite loops.

## 37.5 Production Considerations

Shipping LLM features to production requires solving problems that don't exist in traditional software: output quality is probabilistic, latency is high and variable, costs scale with usage, and failures are often silent (wrong answer, not an error). This section covers the practices that make LLM applications production-grade.

### Evals — Measuring Output Quality

**Evals** are automated tests for LLM output quality. Unlike unit tests (deterministic pass/fail), evals typically score output on a spectrum.

```typescript
// A simple eval suite
interface EvalCase {
  input: string;
  expected: string;
  metric: 'exact_match' | 'contains' | 'llm_judge';
}

const evalCases: EvalCase[] = [
  {
    input: 'What is the capital of France?',
    expected: 'Paris',
    metric: 'contains',
  },
  {
    input: 'Summarize this incident report: ...',
    expected: 'Summary includes: root cause, timeline, action items',
    metric: 'llm_judge', // use another LLM call to score the summary
  },
];

async function runEvals(model: string) {
  const results = await Promise.all(evalCases.map(async (c) => {
    const output = await generateResponse(model, c.input);
    const passed = c.metric === 'contains'
      ? output.toLowerCase().includes(c.expected.toLowerCase())
      : await llmJudge(output, c.expected); // call LLM to score
    return { case: c.input, passed };
  }));

  const passRate = results.filter(r => r.passed).length / results.length;
  console.log(`Pass rate: ${(passRate * 100).toFixed(1)}%`);
  return passRate;
}
```

**Eval-driven development:** write evals *before* changing your prompt, the way you write tests before code. Regression is the main risk when optimizing prompts — a change that fixes one case often breaks another.

### Prompt Caching

Prompt caching (Anthropic, also available in OpenAI batch mode) stores the KV-cache of a prompt prefix on the server. Subsequent requests with the same prefix skip recomputing those tokens — **reducing latency by ~85% and cost by ~90% for cached tokens**.

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

// Mark the expensive, static part of the prompt for caching
const response = await client.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 1024,
  system: [
    {
      type: 'text',
      text: 'You are an expert at analyzing code reviews.',
      cache_control: { type: 'ephemeral' }, // Cache this prefix
    },
    {
      type: 'text',
      // Large static context (e.g. codebase docs, style guide — up to 100K tokens)
      text: largeStaticContext,
      cache_control: { type: 'ephemeral' }, // Cache this too
    }
  ],
  messages: [{ role: 'user', content: dynamicUserQuery }], // Not cached (changes each request)
});

// response.usage.cache_creation_input_tokens — tokens written to cache
// response.usage.cache_read_input_tokens — tokens read from cache (billed at 0.1x)
```

**When to use:** any pattern where a large system prompt or context is static across many requests — RAG pipeline base prompt, a large codebase loaded as context, a detailed persona/instructions.

### Hallucination Mitigation

- **Grounding with RAG** (§37.3) — give the model facts to cite rather than asking it to recall.
- **Explicit "I don't know" instruction** — tell the model to say "I don't have that information" rather than guess.
- **Citations** — require the model to cite the source chunk for each claim; citeable answers are verifiable answers.
- **Temperature 0** — deterministic output reduces variance and hallucination for factual tasks.
- **Output validation** — parse structured output and verify against a schema or business rules.

### Cost and Latency Trade-offs

```
Latency budget:
  p50: model inference ~1–3s for 500-token output
  p99: can be 5–15s under load
  Streaming (server-sent events) masks latency: show first token in ~200–500ms

Cost levers (biggest → smallest):
  1. Model choice (GPT-4o vs GPT-3.5: ~10x cost difference)
  2. Output length (max_tokens limit)
  3. Input length (prompt engineering, RAG chunk count)
  4. Prompt caching (90% reduction on cached prefix)
  5. Batch API (50% discount, hours SLA — for async workloads)
```

**Streaming responses** should be default for any user-facing feature — streaming `text/event-stream` output tokens as they arrive dramatically reduces perceived latency.

### AI Ethics and Responsible Use

LLM-powered features carry ethical responsibilities that traditional software typically doesn't.

- **Bias** — outputs reflect training data biases; can be discriminatory in hiring, lending, content moderation, and healthcare. Audit outputs for demographic disparities; add explicit fairness instructions for high-stakes domains.
- **Transparency** — disclose when users are interacting with AI; never present model output as authoritative human judgement for consequential decisions.
- **Accountability** — log all AI-driven decisions (§Observability), maintain audit trails, and ensure a human can review or override outcomes.
- **Reliability** — validate that the model behaves consistently across inputs; probabilistic output makes silent regression the norm, not the exception.
- **Security** — prompt injection, jailbreaks, and data poisoning are first-class attack surfaces; apply the same rigour as SQL injection (§37.2).

**Practical mitigations at the prompt level:** add explicit bias-reduction instructions ("be impartial; avoid assumptions based on gender, ethnicity, or age"), test with adversarial and demographic edge-case inputs, and validate outputs against your organisation's content policy before surfacing them to users.

### Observability for LLM Apps

Standard app observability (logs/metrics/traces from §26) needs to extend to cover LLM-specific signals.

| What to log/trace | Why |
| --- | --- |
| **Full prompt + response** | Debugging, audit trail |
| **Token counts** (input/output) | Cost attribution |
| **Model + version** | Reproducibility |
| **Latency** (TTFT, total) | SLO tracking |
| **Tool calls + results** | Agent debugging |
| **Eval scores** | Quality regression detection |

**LLM observability platforms:** LangSmith, Langfuse, Helicone, Arize Phoenix — all provide prompt versioning, trace visualization, and cost dashboards. At minimum, log all prompts and responses to your existing logging infrastructure.

## 37.6 Claude & Claude Code

Claude Code is Anthropic's CLI for software engineering. It runs in your terminal, reads your codebase, and autonomously edits files, runs commands, and commits changes — all directed by natural language. It operates as an agentic loop: reason → use a tool → observe the result → repeat. Understanding its extension points (CLAUDE.md, subagents, skills, MCP, hooks) is increasingly relevant as it becomes part of the everyday engineering workflow.

### The explore → plan → code → commit Workflow

The recommended pattern for non-trivial tasks:

1. **Explore** — let Claude read the relevant files and build a picture of the codebase before touching anything
2. **Plan** — ask Claude to produce a written plan; review it and correct misunderstandings before any code changes
3. **Code** — Claude edits files, runs tests, and fixes failures autonomously
4. **Commit** — Claude stages changes and writes a commit message following project conventions

The plan step is the most important: catching a wrong assumption here costs nothing; catching it after 30 edited files is expensive. For non-trivial changes, always review the plan before approving execution.

### Context Management

Claude's context window fills as a session grows; quality degrades near the limit.

| Command | Effect |
| --- | --- |
| `/compact` | Summarises conversation history in place — preserves working state, frees tokens |
| `/clear` | Full reset — use when switching to a completely different task |
| `--continue` | Resumes the last session when starting a new terminal |

As a rule of thumb: compact when context usage reaches ~80%, and always compact before starting a new subtask in a long session.

### CLAUDE.md

CLAUDE.md is the primary customization mechanism — a Markdown file injected into Claude's context at the start of every session.

| Location | Scope |
| --- | --- |
| `~/.claude/CLAUDE.md` | Global (all projects) |
| `<project-root>/CLAUDE.md` | Project-wide |
| `<subdirectory>/CLAUDE.md` | Scoped to that directory tree |

**Good content:** coding conventions, project structure overview, test/build commands, things Claude must never do ("never commit directly to main", "always use the internal logger, not console.log"), domain vocabulary.

**Bad content:** instructions that belong in a single prompt, or information Claude can derive by reading the code itself.

### Subagents

Claude Code can spawn subagents — child Claude instances that work on isolated subtasks in parallel. The orchestrator coordinates; each subagent has its own context window and tool access, so long-running or exploratory work doesn't pollute the main session.

```
Orchestrator
  ├── Subagent A: "explore the auth module and summarise its structure"
  ├── Subagent B: "write unit tests for the CSV parser"
  └── Subagent C: "update the README to reflect the new API shape"
```

Use subagents to parallelise independent tasks and to protect the main context window from exhaustive file searches or large code reads.

### Skills (Slash Commands)

Skills are custom slash commands defined as Markdown files in `.claude/commands/`. Invoking `/skill-name` runs the file's content as a prompt template.

```
.claude/
  commands/
    review.md     →  /review
    refactor.md   →  /refactor
    explain.md    →  /explain
```

A skill file is a plain prompt that can reference `$ARGUMENTS` (the text typed after the slash command name). Skills are version-controlled with the project and shared across the team — they're the right place to encode team-specific workflows that would otherwise require re-prompting from scratch each time.

### MCP (Model Context Protocol)

MCP is Anthropic's open protocol for connecting Claude to external tools and data sources. An MCP server exposes **tools** (callable functions) and/or **resources** (readable data) that Claude can use during a session — think of it as the plugin system for Claude.

```
Claude Code ── MCP client ── MCP server (local or remote)
                               ├── tools:     query_db(), create_issue(), search_web()
                               └── resources: postgres://..., github://repo/...
```

MCP servers are configured in `.claude/settings.json` (project) or `~/.claude/settings.json` (global). Common servers: filesystem, GitHub, PostgreSQL, web search, Slack, Jira.

MCP is the right extension point when Claude needs to interact with an external system as part of its workflow. Skills are just prompt templates; MCP gives Claude live access to real data and actions.

### Hooks

Hooks are shell commands that execute automatically in response to Claude Code lifecycle events — configured in `settings.json`.

| Event | Common use |
| --- | --- |
| `PreToolUse` | Validate or log before Claude calls a tool |
| `PostToolUse` | Run linter/formatter after a file is written |
| `Stop` | Send a desktop notification when Claude finishes |
| `SubagentStop` | Aggregate or post-process subagent output |

```json
// .claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [{ "type": "command", "command": "npm run lint --fix" }]
      }
    ]
  }
}
```

Hooks run in the same shell environment as Claude's bash commands and have access to the same env vars. They're the right tool for automating repetitive hygiene (format on write, test on commit) rather than relying on Claude to remember to do it.

## Generative AI Priority Summary

| Topic | Priority |
| --- | --- |
| **LLM Fundamentals** | |
| Tokens, context windows, context cost | **Critical** |
| Temperature / sampling + stop sequences + frequency/presence penalties | **Important** |
| Model families & capability/cost tradeoffs | **Important** |
| Hallucination, cutoff, lost-in-middle, bias limitations | **Critical** |
| Jailbreaks and data poisoning as attack surfaces | **Know** |
| **Prompting** | |
| Prompt structure components (role, context, instruction, format, constraints) | **Important** |
| Zero-shot vs few-shot | **Critical** |
| Chain-of-thought ("think step by step") | **Deep** |
| Tree-of-Thoughts (parallel branch evaluation) | **Know** |
| Structured output / JSON mode + format variety | **Critical** |
| Iterative prompt refinement cycle | **Important** |
| Prompt injection risks & defenses | **Important** |
| **RAG** | |
| RAG vs fine-tuning tradeoff | **Critical** |
| **Tool Use & Agents** | |
| Tool/function calling (schema, loop) | **Deep** |
| ReAct pattern (reason → act → observe) | **Deep** |
| Guardrails (least privilege, human-in-loop, max iterations) | **Important** |
| When agents vs single-shot | **Critical** |
| **Production** | |
| AI ethics: bias, transparency, accountability | **Important** |
| Evals (write before optimizing prompts) | **Deep** |
| Prompt caching (latency + cost reduction) | **Important** |
| Hallucination mitigation (grounding, citations) | **Deep** |
| Streaming responses (perceived latency) | **Important** |
| Observability (log prompts + tokens + scores) | **Important** |
| **Claude Code** | |
| explore → plan → code → commit workflow | **Important** |
| CLAUDE.md (global vs project vs directory scope) | **Important** |
| Subagents (parallel isolation, context protection) | **Deep** |
| Skills / slash commands (`.claude/commands/`) | **Important** |
| MCP — tools vs resources, config in settings.json | **Deep** |
| Hooks — PostToolUse, Stop, event-driven automation | **Know** |

---

_End of Part 9. Continue to **Part 10** (Leadership & Soft Skills) in [`10_Leadership_38.md`](./10_Leadership_38.md), or return to the [README](./README.md)._
