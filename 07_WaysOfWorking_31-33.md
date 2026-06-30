# Ways of Working (25, 31-33)
### Git & Version Control, Engineering Excellence, Agile & Delivery, Documentation

---

# 25. Git & Version Control

Version control is infrastructure — most engineers use it daily but few can explain the trade-offs between branching strategies, why `--force-with-lease` is safer than `--force`, or what goes wrong when you rebase a public branch. Interviews at senior level probe the *why* behind common practices: *why trunk-based over GitFlow for a team running continuous deployment? Why squash merge for feature branches?* This section covers the strategies, merge mechanics, commit conventions, and PR discipline that separate a seasoned engineer from someone who just knows `git add / commit / push`. It also covers the tooling (conventional commits, semantic-release, commitizen) that makes commit history machine-readable and release notes automatic.

## 25.1 Branching Strategies

The choice of branching strategy determines how frequently integration happens, how much merge pain teams tolerate, and whether CI/CD is a first-class workflow or an afterthought.

### Trunk-Based Development (TBD)

The strategy recommended by the DORA research as the foundation of high-performing engineering teams. All developers integrate to a single long-lived branch (`main` / `trunk`) at least once per day.

**Core requirements:**
- **Short-lived branches** — feature branches exist ≤ 1–2 days; anything longer is a smell.
- **Feature flags** — incomplete features ship behind a flag; the flag gate is what keeps trunk deployable (see §31.6).
- **Comprehensive CI** — every push runs the full test suite; a red build is the team's top priority.
- **Pair programming / small PRs** — small changes make the daily-integration cadence realistic.

**Why it works:**
- Continuous integration = continuous conflict resolution. You never accumulate a week's worth of divergent changes to reconcile.
- DORA data: teams on TBD ship ~200× more frequently with 24× faster recovery times than teams on long-lived branches.
- Forces good habits: small PRs, feature flags, fast tests. The strategy is a forcing function.

**When it doesn't work well:**
- Open-source projects with untrusted contributors (need pull-request review gates).
- Very large monorepos with slow CI (need build/test impact analysis to keep trunk green).
- Regulated environments that require a named release branch with a formal sign-off window — use a short-lived release branch cut from trunk, not a long-lived one.

```text
Trunk-Based Development:

         o──o──o──o──o──o   trunk (main) — always deployable
              │    │
              │    └─ o─o   feature/checkout (< 2 days, feature-flagged)
              └─ o─o        fix/login-timeout (< 1 day)
```

**Interview answer framework:** "We use trunk-based development: all engineers integrate at least daily, incomplete work ships behind a feature flag. This keeps trunk deployable at all times and eliminates merge debt. Our CI runs in under 10 minutes; a broken build is P0 for the team."

### GitFlow

Introduced by Vincent Driessen (2010). Suited for **scheduled release cycles** (mobile apps, libraries, enterprise software with versioned releases). Heavy overhead — generally a poor fit for web services with continuous deployment.

**Branch model:**

```text
GitFlow branch model:

main      ──────────────────────────────────────────── (tagged releases only)
            ↑                    ↑           ↑
           tag v1.0            tag v1.1    tag v2.0
             │                   │
develop   ──○──○──○──○──○──○──○──○──○──○──○──○──── (integration branch)
               │         │         │
feature/  ─────○──○──○──┘         │         (merges back to develop)
feature/            ─────○──○──○──┘
                               │
release/  ──────────────────────○──○──○──────────── (stabilisation + bugfixes)
                                            │
hotfix/   ──────────────────────────────────○──○──── (merges to main + develop)
```

**Branch purposes:**

| Branch | Lifetime | Purpose |
|---|---|---|
| `main` | permanent | production code only; tagged with version numbers |
| `develop` | permanent | integration branch; always ahead of main |
| `feature/*` | short | new work; branches from develop, merges back to develop |
| `release/*` | short | stabilisation; no new features, only bug fixes; merges to main + develop |
| `hotfix/*` | very short | production emergency; branches from main, merges to main + develop |

**GitFlow lifecycle:**
1. Branch `feature/checkout` from `develop`
2. Work, commit, open PR → merge to `develop`
3. When sprint ends: cut `release/1.2` from `develop`
4. QA / stabilisation commits on release branch
5. Merge `release/1.2` → `main` (tag `v1.2.0`) and → `develop`
6. Production bug: cut `hotfix/1.2.1` from `main`, fix, merge → `main` (tag) and → `develop`

**Trade-offs:**

| Aspect | GitFlow | TBD |
|---|---|---|
| Release model | Scheduled, versioned | Continuous |
| Merge complexity | High (two long-lived branches) | Low |
| Feature isolation | Excellent | Via feature flags |
| CI/CD fit | Poor (long integration lag) | Excellent |
| DORA performance | Correlates with lower | Correlates with higher |

**When GitFlow makes sense:** mobile SDK releases, npm packages, firmware, enterprise products with customer-negotiated release dates and a defined QA window.

### GitHub Flow

Simplified: `main` + short-lived feature branches. Feature branch → PR → code review → merge to main → deploy. No `develop`, no release branches. Closest to TBD without requiring strict daily integration. The default for most web product teams that haven't fully adopted TBD.

## 25.2 Merge Strategies

How branches land on `main` shapes the history and archaeology experience.

| Strategy | History shape | `git blame` | Bisect-ability | Recommendation |
|---|---|---|---|---|
| **Merge commit** | Non-linear; preserves full branch history | Accurate to author | Good | Monorepos, open-source (clear attribution) |
| **Squash merge** | Linear; one commit per PR | All blame to squash committer | Excellent | Feature branches where in-progress commits are noise |
| **Rebase merge** | Linear; all branch commits preserved | Accurate | Excellent | When branch commits are clean and meaningful |

**Recommendation for most teams:** squash feature branches (clean history, easy `git log`), use merge commits for release branches (preserves the merge point for auditing).

## 25.3 Rebase & History Rewriting

```bash
# Interactive rebase — rewrite last 4 commits before push
git rebase -i HEAD~4
# pick / squash / reword / fixup / drop each commit

# Amend the last commit message (before push only)
git commit --amend --no-edit

# Rebase feature branch onto latest main (avoid merge commits)
git fetch origin
git rebase origin/main
```

**Golden rule:** never rebase commits that have been pushed to a shared remote branch. Rewriting shared history requires force-push and causes `rejected (non-fast-forward)` errors for everyone on the branch.

**`--force-with-lease` over `--force`:**
```bash
git push --force-with-lease   # safe: fails if remote has new commits you haven't fetched
git push --force               # unsafe: silently overwrites any new upstream commits
```
`--force-with-lease` prevents accidentally clobbering a teammate's commits on the same branch.

## 25.4 Conventional Commits & semantic-release

Conventional Commits is a lightweight specification for commit message structure that makes history machine-readable and enables automated semantic versioning.

**Format:**
```
<type>(<scope>): <short description>

[optional body]

[optional footer: BREAKING CHANGE: ...]
```

**Type → SemVer mapping:**

| Type | SemVer bump | Use for |
|---|---|---|
| `feat` | minor | new feature |
| `fix` | patch | bug fix |
| `perf` | patch | performance improvement |
| `BREAKING CHANGE` (footer) | major | any breaking API change |
| `docs`, `chore`, `refactor`, `test`, `ci` | none | maintenance |

```bash
# Good examples
git commit -m "feat(auth): add OAuth 2.0 PKCE flow"
git commit -m "fix(api): handle null pagination cursor"
git commit -m "feat!: remove legacy v1 API endpoints"
# The ! is shorthand for BREAKING CHANGE

# Bad examples (no type, no value to tooling)
git commit -m "fix stuff"
git commit -m "WIP"
```

**semantic-release CI setup:**
```yaml
# .github/workflows/release.yml
- name: Release
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
  run: npx semantic-release
# semantic-release reads commits since last tag, determines version bump,
# generates CHANGELOG.md, tags the release, and publishes to npm — all automated.
```

**commitizen** — interactive commit helper that enforces the format at commit time:
```bash
npm install -D commitizen cz-conventional-changelog
# Then: git cz   (instead of git commit)
# Prompts: type → scope → description → breaking change?
```

**Why this matters in interviews:** "We use conventional commits with semantic-release in CI. It removes the human decision of 'is this a minor or patch bump?' from the release process and keeps CHANGELOG.md honest. The format also makes `git log --oneline` parseable at a glance."

## 25.5 PR Best Practices

Small, focused pull requests are the single biggest lever on review quality and merge safety.

### PR size discipline

| PR size | Lines changed | Review time | Defect escape rate |
|---|---|---|---|
| Small | < 200 | 15–30 min | Low |
| Medium | 200–500 | 45–90 min | Medium |
| Large | 500–1000 | 2–4 h | High |
| Huge | > 1000 | Often rubber-stamped | Very high |

**Research finding (SmartBear, 2010 / replicated): reviewers can effectively inspect ~400 lines per hour; above ~500 changed lines in a single review, defect detection rates fall sharply.**

Strategies for keeping PRs small:
- **Feature flags** — land the dead code path first, turn it on later (see §31.6).
- **Stacked PRs / branch trains** — PR1 = data model, PR2 = service layer, PR3 = API, PR4 = UI. Each is reviewable independently.
- **Preparatory refactors** — separate PR for rename/restructure; feature PR only for the new logic.

### PR description template

```markdown
## What
<!-- 1-2 sentences: what does this change do? -->

## Why
<!-- Business or technical context; link to ticket -->

## How
<!-- Non-obvious implementation choices; what to look at first -->

## Testing
<!-- What you tested manually; which tests cover this -->

## Checklist
- [ ] Tests added/updated
- [ ] No secrets in code
- [ ] Breaking change? (API/schema/contract)
```

### Review etiquette (author + reviewer)

**Author:**
- Keep PRs under 400 changed lines where possible.
- Walk reviewers through any non-obvious design in the description.
- Respond to all comments before re-requesting review.
- Don't take review comments personally — they're about the code, not you.

**Reviewer:**
- Review within one business day (unreviewed PRs block the whole team).
- Distinguish blocking (`must fix`) from non-blocking (`nit:` or `suggestion:`).
- Ask questions rather than making demands ("Why did you choose X over Y?" rather than "Use Y instead").
- Approve and merge once the must-fix items are addressed — don't block on nits after an LGTM.
- Approve the PR even if you'd have done it differently; your job is to catch defects and risks, not to enforce personal style.

## 25.6 Essential Git Commands

Commands that come up in interviews or day-to-day that go beyond the basics:

```bash
# Stash uncommitted work (including untracked files)
git stash push -u -m "WIP: half-done auth refactor"
git stash list
git stash pop        # reapply and drop
git stash apply stash@{2}  # reapply without dropping

# Cherry-pick a single commit to current branch
git cherry-pick <commit-sha>
git cherry-pick A^..B   # range A to B (inclusive)

# Reflog — the safety net for "I accidentally deleted that commit"
git reflog              # shows every HEAD movement, including rebases and resets
git checkout HEAD@{3}   # return to state 3 moves ago
git branch recover-work HEAD@{3}  # create a branch from reflog entry

# Bisect — binary search for the commit that introduced a bug
git bisect start
git bisect bad           # current commit is broken
git bisect good v1.2.0   # last known-good tag
# Git checks out midpoint; you test and mark:
git bisect good   # or:
git bisect bad
# Repeat until: "abc1234 is the first bad commit"
git bisect reset # return to HEAD

# Blame with line-range and ignore whitespace
git blame -L 50,80 -w src/auth/jwt.ts

# Find which commit deleted a function
git log -S "functionName" --all --oneline
```

## Git & Version Control Priority Summary

| Topic | Priority |
|---|---|
| **Trunk-Based Development** — requirements, why DORA prefers it, when it doesn't work | **Critical** |
| **GitFlow** — full branch model, lifecycle (feature→develop→release→main+hotfix), trade-offs vs TBD | **Deep** |
| **GitHub Flow** — simplified model, relationship to TBD | **Important** |
| **Merge strategies** — merge commit vs squash vs rebase; recommendation | **Deep** |
| **Rebase & `--force-with-lease`** — golden rule, interactive rebase | **Important** |
| **Conventional Commits** — format, type→SemVer table, semantic-release CI | **Deep** |
| **commitizen** — interactive commit helper | **Learn** |
| **PR size discipline** — 400-line research finding, strategies for small PRs | **Critical** |
| **PR description template** — what/why/how/testing/checklist | **Deep** |
| **Review etiquette** — author + reviewer responsibilities | **Important** |
| **Stash / cherry-pick** — syntax and use cases | **Important** |
| **Reflog** — safety net, recovering lost commits | **Deep** |
| **Bisect** — binary search workflow | **Learn** |

---

# 31. Engineering Excellence

At the senior level, interviews stop being purely about whether you can write the code and start probing whether you can be trusted to *own* it: do your changes land safely, are they reviewable, do they degrade gracefully, and can the team move faster because of how you work rather than slower? "Engineering excellence" is the umbrella term for the practices that make a team's output predictable and its codebase pleasant to live in — code craftsmanship, review discipline, a testing strategy that catches the right bugs at the right cost, delivery metrics that tell you whether you're actually improving, and the operational maturity to handle incidents without blame. None of it is glamorous, and that's exactly why interviewers use it to separate seniors from mid-levels: *can you articulate why a 200-line PR is easier to ship than a 2,000-line one, and back it with a metric?* This section covers the craft, the measurements, and the culture.

## 31.1 Code Craftsmanship & Review

Craftsmanship is the day-to-day discipline that keeps a codebase changeable. The principles are old (Kent Beck, "Uncle Bob" Martin) but the interview value is in *applying* them, not reciting them.

**Clean-code fundamentals (the parts that survive contact with reality):**

- **Names reveal intent.** `daysSinceLastLogin` beats `d`. A good name removes the need for a comment.
- **Small functions, one level of abstraction.** A function should do one thing; if you need "and" to describe it, split it.
- **Boy-scout rule** — leave code a little cleaner than you found it, but *scope the cleanup to the change* (don't smuggle a refactor into a bugfix PR; it muddies review and `git blame`).
- **Comments explain *why*, not *what*.** Well-named code already says what it does; reserve comments for non-obvious constraints, invariants, and workarounds.

### Code review

The single highest-leverage quality practice, and a frequent interview topic ("walk me through how you review a PR").

```text
What to look for, in priority order:
1. Correctness      — does it do what it claims? edge cases, error paths?
2. Security         — injection, authz, secrets, unsafe deserialization
3. Design           — does it fit the architecture? right abstraction level?
4. Readability       — will the next person understand it in 6 months?
5. Tests            — do they cover the change and fail for the right reasons?
6. Style/nits       — leave to linters/formatters, not humans
```

**Keep PRs small.** Review quality drops sharply past ~400 lines; reviewers start rubber-stamping. Small PRs get faster, deeper review and isolate `git bisect` blast radius. **Review etiquette:** critique the code, not the author; distinguish blocking concerns from suggestions (prefix nits with "nit:"); approve with comments rather than blocking on taste. Automate everything mechanical (formatting, lint, import order) so humans only discuss what humans are good at.

### Definition of Done & quality gates

A shared **Definition of Done (DoD)** prevents "done" from meaning "compiles on my machine." A typical DoD: code reviewed and merged, tests written and green, docs/changelog updated, feature-flagged or deployed, acceptance criteria met. **Automated quality gates** enforce the mechanical parts in CI — lint passes, type-check passes, coverage doesn't drop below a threshold, no new high-severity vulnerabilities, bundle size within budget. The gate is the wall; the DoD is the checklist (see §24 for pipeline mechanics).

### Managing technical debt

Debt is not inherently bad — it's a loan. The skill is being **deliberate** about it. Martin Fowler's **technical-debt quadrant** classifies debt on two axes:

| | **Reckless** | **Prudent** |
| --- | --- | --- |
| **Deliberate** | "We don't have time for design" | "Ship now, refactor next sprint — and we tracked it" |
| **Inadvertent** | "What's layering?" | "Now we know how it should have been done" |

Prudent-deliberate debt is a legitimate engineering decision *if it's tracked* (a ticket, a `// TODO(JIRA-123)`) and paid down. Strategies: allocate a fixed fraction of each sprint to paydown, use the boy-scout rule for opportunistic cleanup, and make debt visible to product so it's a shared trade-off, not a hidden tax.

**Interview framing:** "How do you handle tech debt on a team that's always shipping?" — Make it *visible and quantified* (link it to slowed delivery or incident frequency), classify deliberate vs reckless, and negotiate a steady paydown budget rather than a doomed "big rewrite." A rewrite is almost always the wrong answer; incremental strangler-fig migration (§36.3) is usually right.

## 31.2 Testing Strategy & Culture

Tooling lives in §15 (Vitest/Playwright/RTL/MSW); this is the *strategy* lens — what to test, at what level, and why. The classic model is the **test pyramid**: many fast unit tests at the base, fewer integration tests in the middle, very few slow end-to-end tests at the top.

```text
        /\        E2E        few, slow, brittle, high-confidence (full user flows)
       /  \
      /----\     Integration  some, medium speed (module + DB/API boundaries)
     /      \
    /--------\   Unit         many, fast, isolated (pure logic, branches)
```

**Why the shape matters:** E2E tests give the most confidence but are slow and flaky, so a top-heavy "ice-cream cone" suite is expensive and unreliable. Push coverage down to the cheapest level that still catches the bug. The modern nuance is the **testing trophy** (Kent C. Dodds) — for frontend/integration-heavy code, weight *integration* tests most heavily, since they exercise real behavior without E2E's cost.

- **TDD (red → green → refactor)** — write a failing test, make it pass minimally, then refactor under green. Best for well-specified logic (parsers, pricing rules); less dogmatically applied to exploratory UI work.
- **Coverage is a signal, not a target.** 100% coverage proves lines *ran*, not that behavior is *correct*. Chasing a coverage number produces assertion-free tests. Use it to find untested branches, not as a KPI (Goodhart's law).
- **Shift-left** — catch defects as early as possible (type system, lint, unit tests, pre-merge CI) where they're cheapest to fix, rather than in QA or production.
- **Contract testing** (e.g., Pact) verifies that a service and its consumers agree on an API shape, catching integration breakage without a full E2E environment.

**Interview framing:** "What's your testing strategy?" — Lead with the pyramid/trophy and *cost vs confidence*: test business logic exhaustively at the unit level, test boundaries (DB, HTTP) at the integration level, and reserve a handful of E2E tests for critical revenue paths (login, checkout). Mention flaky-test hygiene (quarantine, don't ignore) — flaky tests erode trust in the whole suite.

### Test types taxonomy

Not all tests are equal — and interviews use overlapping vocabulary that's worth knowing precisely:

| Type | What it verifies | Speed | Owner |
| --- | --- | --- | --- |
| **Unit** | Single class/function in isolation | ~1 ms | Developer |
| **Integration / component** | Module + its infrastructure boundary (DB, HTTP) | Seconds | Developer |
| **E2E** | Full user flow through real browser or HTTP | Minutes | Developer / QA |
| **Smoke** | Critical paths only — does the app start and respond? | Seconds | CI/CD pipeline |
| **Regression** | Full suite — did anything that used to work break? | Minutes–hours | CI/CD / QA |
| **Acceptance / UAT** | Meets business requirements as specified | Variable | Product / QA |
| **Exploratory** | Unscripted, investigative — find what automation misses | Variable | QA |

**Smoke vs regression:** Smoke tests are fast and narrow — "the login endpoint returns 200 and the homepage renders." Run them after every deploy to catch bad deploys within seconds. Regression tests are wide and slow — they run the full suite to ensure nothing that used to work is now broken. Run them on a longer cadence (nightly, pre-release) rather than blocking every deploy.

**Sanity vs smoke:** Sanity testing is a quick check after a bug fix — "does this specific fix work?" — narrower even than smoke. The two terms are often used interchangeably; what matters is the principle: after a change, verify the changed area before running the full regression.

### TDD (Test-Driven Development)

TDD is a development discipline, not a test type. The cycle: **red → green → refactor**.

1. **Red** — write a failing test for the smallest piece of behaviour you want to add. It must fail for the right reason (the behaviour doesn't exist), not because of a compilation error.
2. **Green** — write the *minimal* code to make the test pass. Don't generalise yet; just make it green.
3. **Refactor** — clean up the implementation while keeping all tests green. The test suite is now the safety net.

```typescript
// Red: write the failing test first
it("returns 0 for an empty cart", () => {
  const cart = new ShoppingCart();
  expect(cart.total()).toBe(0);   // ← fails: ShoppingCart doesn't exist yet
});

// Green: minimal implementation
class ShoppingCart {
  total() { return 0; }          // ← just enough to pass
}

// Red again: next behaviour
it("sums item prices", () => {
  const cart = new ShoppingCart();
  cart.add({ price: 10 });
  cart.add({ price: 20 });
  expect(cart.total()).toBe(30); // ← fails: no add(), no real sum
});

// Green: extend minimally
class ShoppingCart {
  private items: { price: number }[] = [];

  add(item: { price: number }) { this.items.push(item); }
  total() { return this.items.reduce((s, i) => s + i.price, 0); }
}
// Refactor: rename, extract, tighten — tests stay green throughout
```

TDD works best for well-specified logic (parsers, pricing rules, state machines). Apply it pragmatically: writing tests after is better than no tests; writing them first is better when requirements are clear. The main benefit is design feedback — code that's hard to test under TDD is usually also hard to change.

### BDD (Behaviour-Driven Development) and Cucumber

BDD extends TDD with a shared language — **Gherkin** (Given/When/Then) — that business stakeholders, QA, and developers can all read. Scenarios become living documentation that doubles as automated acceptance tests.

```gherkin
# features/shopping-cart.feature
Feature: Shopping cart

  Scenario: Customer adds an item
    Given the cart is empty
    When the customer adds "Book" costing £10
    Then the cart total should be £10
    And the cart should contain 1 item

  Scenario: Discount is applied for members
    Given the customer is a member
    When the customer adds "Book" costing £100
    Then the cart total should be £90
```

```typescript
// step-definitions/cart.steps.ts
import { Given, When, Then } from "@cucumber/cucumber";
import { ShoppingCart } from "../../src/cart";
import assert from "assert";

let cart: ShoppingCart;
let isMember = false;

Given("the cart is empty", () => { cart = new ShoppingCart(); });
Given("the customer is a member", () => { isMember = true; cart = new ShoppingCart({ member: true }); });

When("the customer adds {string} costing £{int}", (_name: string, price: number) => {
  cart.add({ price });
});

Then("the cart total should be £{int}", (expected: number) => {
  assert.strictEqual(cart.total(), expected);
});
```

Cucumber is valuable when QA or product writes or reviews scenarios — it bridges requirements and tests. For developer-only test suites, plain `describe`/`it` with descriptive names achieves the same readability with less tooling overhead.

### Performance Testing

Performance testing verifies the system behaves correctly under load — before that load reaches production. The three main profiles:

| Type | What it simulates | Goal |
| --- | --- | --- |
| **Load test** | Expected production traffic | Verify latency/throughput stays within SLOs |
| **Stress test** | Traffic beyond expected peak | Find the breaking point and failure mode |
| **Soak test** | Sustained normal load over hours | Catch memory leaks, connection pool exhaustion |

**k6** is the standard tool for backend load testing — tests are JavaScript, executed as Go binaries, with built-in threshold assertions.

```javascript
// k6/load-test.js
import http from "k6/http";
import { check, sleep } from "k6";

export const options = {
  stages: [
    { duration: "30s", target: 20 },  // ramp up to 20 virtual users
    { duration: "60s", target: 20 },  // hold at 20 VUs
    { duration: "10s", target: 0 },   // ramp down
  ],
  thresholds: {
    http_req_duration: ["p(95)<200"],  // 95th percentile < 200 ms
    http_req_failed:   ["rate<0.01"],  // < 1% error rate
  },
};

export default function () {
  const res = http.get("https://staging.example.com/api/users");
  check(res, { "status is 200": (r) => r.status === 200 });
  sleep(1);
}
```

```bash
k6 run k6/load-test.js
# Output: VUs, iterations, p50/p95/p99 latency, error rate
```

Run load tests in CI against staging before high-traffic releases; run stress tests ad-hoc when planning capacity; schedule soak tests weekly if memory leaks are a risk (long-running Node processes, caches that grow unbounded).

### Penetration Testing (basics)

Penetration testing (pentest) is systematic, authorised attempt to exploit security vulnerabilities — distinct from automated security scanning:

| | Security scanning | Penetration testing |
| --- | --- | --- |
| **How** | Automated tools | Manual + tools |
| **Finds** | Known CVEs, dependency vulnerabilities | Logic flaws, chained exploits, business-layer flaws |
| **Cadence** | Every CI run | Quarterly or before major releases |
| **Who** | Developer / DevSecOps | Security engineer or external firm |

**OWASP ZAP (Zed Attack Proxy)** is the standard open-source tool for automated DAST (Dynamic Application Security Testing) — it proxies requests, spiders the application, and reports on OWASP Top 10 categories (see §27 for the full list). Run it in CI against a staging environment as a baseline gate:

```bash
# Docker-based ZAP baseline scan (passive — doesn't attack, just reports)
docker run --rm ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py \
  -t https://staging.example.com \
  -r zap-report.html \
  -I  # exit 0 even with alerts (use -I to not block CI initially)
```

As a senior engineer you're unlikely to run full manual pentests yourself, but you should: understand the OWASP Top 10 (§27), review pentest reports and translate findings into properly prioritised tickets, and ensure deployment pipelines include at minimum an automated DAST scan against staging before production releases.

## 31.3 Delivery Metrics (DORA)

"Are we getting better?" needs a number. The **DORA metrics** (from Google's DevOps Research and Assessment / *Accelerate*) are the industry-standard four, and research shows they correlate with both delivery speed *and* stability — they are not a trade-off.

| Metric | Measures | Elite | Low |
| --- | --- | --- | --- |
| **Deployment frequency** | How often you ship to prod | On-demand (multiple/day) | < monthly |
| **Lead time for changes** | Commit → running in prod | < 1 hour | > 6 months |
| **Change failure rate** | % of deploys causing a failure | 0–15% | 40–60% |
| **MTTR** (time to restore) | How fast you recover from failure | < 1 hour | > 1 week |

The first two measure **throughput**, the last two measure **stability**. The key insight from *Accelerate*: elite teams score well on *both* — speed and safety reinforce each other when you have small batches and strong automation. (A fifth metric, **reliability**, was later added.)

**Practices that move these numbers:**

- **Trunk-based development** — short-lived branches merged to `main` daily, behind feature flags. Reduces merge hell and lead time (contrast with long-lived GitFlow branches; see §25.1).
- **Feature flags** — decouple *deploy* from *release*. Ship dark, enable for 1% of users, ramp up. Lets you deploy frequently without exposing half-finished work, and gives an instant kill-switch (lowering MTTR).
- **Progressive delivery** — **canary** (route a small % of traffic to the new version, watch metrics, then ramp) and **blue-green** (two identical environments, flip the load balancer; instant rollback by flipping back). Both shrink change-failure blast radius and MTTR.

**Interview framing:** "How do you know your team is improving?" — Cite DORA, and stress that frequency and stability rise *together* through small batches, automation, and flags — not by slowing down to be "safe."

## 31.4 Reliability & Incident Management

Production *will* break; maturity is measured by how you respond. Note this builds on the SLI/SLO/SLA and error-budget definitions in **§26.2** — those are the *targets*; this is the *response*.

### The incident lifecycle

```text
Detect → Triage → Mitigate → Resolve → Learn
  │         │         │          │         │
 alert    assess   stop the    fix root  blameless
 fires    severity  bleeding    cause     postmortem
          assign    (rollback,            + action
          IC        flag off)             items
```

**Mitigate before you fix.** The first goal is to *stop user pain* — roll back, flip a feature flag, fail over — not to find the root cause. Root-causing happens after the bleeding stops.

**Severity levels** give a shared vocabulary for urgency:

| Sev | Impact | Response |
| --- | --- | --- |
| **SEV1** | Major outage, revenue/data loss | All-hands, page immediately, incident commander |
| **SEV2** | Significant degradation, partial outage | Page on-call, active mitigation |
| **SEV3** | Minor / single-feature, workaround exists | Business-hours fix |

**On-call & the incident commander (IC).** A sustainable rotation (reasonable frequency, follow-the-sun for global teams, comp/time-off) prevents burnout. During a SEV1, a single **IC** coordinates — they don't fix; they delegate, communicate to stakeholders, and keep a timeline.

### Blameless postmortems

After resolution, write a **blameless postmortem**: timeline, impact, root cause(s), what went well, what didn't, and **action items with owners and due dates**. "Blameless" is the load-bearing word — the premise is that people act reasonably given the information they had, so you fix *systems and processes*, not people. Blame drives incidents underground; blamelessness surfaces them. Use **5 Whys** to push past symptoms to systemic causes ("the deploy failed" → … → "we have no staging parity").

**Error budgets** tie it together: if your SLO is 99.9% (§26.2), you have a budget of ~43 min/month of downtime. Spend it on shipping features; when it's exhausted, freeze risky changes and prioritize reliability. It turns "move fast" vs "be stable" from an argument into a number.

**Interview framing:** "Walk me through a production incident you handled." — Use the lifecycle (detect → mitigate → resolve → learn), emphasize *mitigation first*, and land on the blameless postmortem with a concrete systemic fix you drove. That arc signals seniority more than the technical detail of the bug.

## 31.5 Behavioral Interviews & STAR

Behavioral interviewing is how senior engineers are assessed on non-technical dimensions: how they handle conflict, deliver under ambiguity, mentor others, make architectural trade-offs under constraints, and influence without authority. For a senior role, 30–40% of interview time is typically behavioral. The STAR format gives you a reliable structure for answering "tell me about a time when..." questions concisely and in a way that signals seniority rather than just competence. This section covers the framework, the themes you must have stories for, and the leveling signals that separate a senior answer from a mid-level one.

### The STAR framework

**S**ituation → **T**ask → **A**ction → **R**esult

| Element | What to cover | Target length |
| --- | --- | --- |
| **Situation** | Context: team, product, constraint or trigger | 2–3 sentences |
| **Task** | Your specific responsibility in that situation | 1–2 sentences |
| **Action** | What *you* specifically did — concrete, first-person | 3–5 sentences (the majority) |
| **Result** | Measurable outcome + what you learned or changed | 2–3 sentences |

The most common mistake is spending 60% of the answer on Situation and Task, leaving only a vague "we shipped it" for Action and Result. Interviewers want your decision-making — the background is just scaffolding.

### Must-have stories for a senior role

Prepare at least one story per theme. The questions below are essentially universal — they appear at every company regardless of size or stack:

| Theme | Typical question |
| --- | --- |
| **Technical leadership** | "Tell me about a significant technical decision you made, especially one others disagreed with." |
| **Conflict resolution** | "Tell me about a time you disagreed with a colleague, PM, or manager." |
| **Ambiguity** | "Describe a time you had to move forward without all the information you needed." |
| **Mentoring** | "How have you helped more junior engineers grow?" |
| **Failure & learning** | "Tell me about a project that didn't go as planned. What did you do?" |
| **Scope & trade-offs** | "Tell me about a time you had to cut scope to ship on time." |
| **Cross-team influence** | "Tell me about a time you drove change across teams without direct authority." |
| **Production incident** | "Walk me through a production incident you led." |

### Example: Technical leadership — anti-pattern vs good

**Anti-pattern:** *"We had a monolith. I researched microservices. We migrated. Things got faster."*
No conflict, no decision process, no adversity, no measurable result — this is a resume bullet, not a story.

**Good STAR:**
- **S:** "Our checkout service was owned by three teams who all deployed together — any bug in payments blocked marketing promotions from shipping, which happened twice in one quarter."
- **T:** "I was the tech lead for the payments team and was asked to recommend an architecture path that reduced deployment coupling."
- **A:** "I ran a two-day spike to prototype the strangler-fig pattern — extracting one bounded context while keeping the rest of the monolith intact. I wrote an ADR documenting the trade-offs: deploy independence versus distributed tracing overhead and network latency. I presented both teams with the ADR and the spike results, proposed starting with just the payments context as a bounded pilot, and offered to own the migration myself so neither team took on unplanned work."
- **R:** "We shipped the extracted service in six weeks. Checkout deploys went from requiring three-team sign-off to autonomous. Three months later the other teams requested the same migration for their contexts. The ADR format we used became the team's template for future architecture decisions."

The action is specific, first-person, involves concrete artefacts (ADR, spike, strangler-fig), and the result is measurable and has a downstream effect.

### Leveling signals — what separates senior answers

| Signal | Mid-level answer | Senior answer |
| --- | --- | --- |
| **Scope** | Fixed my own code | Changed how the team works |
| **Conflict** | Avoided or deferred | Named it, engaged directly, found shared ground |
| **Failure** | "I learned to communicate better" | Specific process change you drove afterwards |
| **Trade-offs** | Chose option A | Named options B and C and why they were rejected |
| **Influence** | Got approval from manager | Built consensus across teams, earned buy-in |
| **Result** | "It went well" | Metrics: time saved, error rate, deploy frequency |

### What to avoid

- **"We" throughout the Action** — share credit in Result, but the interviewer needs to know *your* specific contribution.
- **No adversity** — stories where everything went smoothly don't demonstrate resilience or judgement under pressure.
- **Vague results** — "the team was happier" is not a result; "incident frequency dropped by half in the following quarter" is.
- **Over-long setup** — if you're 3 minutes in and still explaining the background, you've lost the room.

### Questions to ask interviewers

These signal senior-level thinking and give you real signal about the role and culture:

1. *"What does a successful first six months look like — and how is that measured?"* (reveals whether the role is well-defined)
2. *"How does the eng org handle postmortems? Can you walk me through a recent one?"* (reveals blameless vs blame culture)
3. *"What's the biggest technical challenge the team is facing right now?"* (lets you demonstrate domain depth)
4. *"How are architectural decisions made — is there an RFC or ADR process?"* (reveals decision-making maturity)
5. *"What does the on-call rotation look like, and what's typical alert volume?"* (reveals operational health)

## 31.6 Feature Flags

Feature flags (also: feature toggles, feature switches) are a **deployment decoupling mechanism**: code ships to production dark; the flag controls whether users see it. This separates the *deploy* event from the *release* event — a distinction central to trunk-based development, progressive delivery, and low MTTR. Flags let you merge continuously without long-lived branches, ramp up a risky change to 1% of users before 100%, and roll back instantly without a deploy. This section covers flag types, the three main platforms, targeting rules, and the flag debt lifecycle problem that bites every team.

### Flag types

| Type | Description | Example use |
| --- | --- | --- |
| **Release (boolean)** | On / off globally | Roll out a new feature |
| **User / cohort targeting** | On/off per user, team, or segment | Beta users, internal staff |
| **Percentage rollout** | Gradual ramp: 0% → 5% → 50% → 100% | Canary release |
| **Multivariate** | Multiple variants (A/B/C) | UI experiment, pricing test |
| **Kill switch** | Off unless emergency | Instant rollback without redeploy |
| **Ops / config flag** | Long-lived config value | Timeout values, rate limits, feature caps |

**Ops flags are fundamentally different from release flags.** Ops flags live forever; release flags are temporary. Mixing them without clear categorisation is the main source of flag debt.

### LaunchDarkly

LaunchDarkly is the dominant commercial platform. The SDK evaluates flags **locally from a streaming update** (~0 ms latency per evaluation) — your app doesn't make a round-trip on each request.

```typescript
import * as ld from '@launchdarkly/node-server-sdk';

const ldClient = ld.init(process.env.LD_SDK_KEY!);
await ldClient.waitForInitialization();

const context: ld.LDContext = {
  kind: 'user',
  key: user.id,
  email: user.email,
  plan: user.plan,       // custom attributes drive targeting rules
  country: user.country,
};

const enabled = await ldClient.variation('new-checkout-flow', context, false);
```

```tsx
// React SDK
import { useFlags } from 'launchdarkly-react-client-sdk';

function CheckoutButton() {
  const { 'new-checkout-flow': newCheckout } = useFlags();
  return newCheckout ? <NewCheckout /> : <LegacyCheckout />;
}
```

### OpenFeature — the vendor-neutral standard

[OpenFeature](https://openfeature.dev/) is a CNCF project defining a **vendor-agnostic SDK interface**. You write application code against the OpenFeature API and swap providers (LaunchDarkly, Unleash, Flagsmith, DevCycle) by changing a single configuration line — no changes to flag call sites.

```typescript
import { OpenFeature } from '@openfeature/server-sdk';
import { LaunchDarklyProvider } from '@openfeature/launchdarkly-provider';

// Swap to Unleash or any provider without changing application code
OpenFeature.setProvider(new LaunchDarklyProvider(sdkKey));

const client = OpenFeature.getClient();
const enabled = await client.getBooleanValue(
  'new-checkout-flow',
  false,                             // default value
  { targetingKey: user.id, email: user.email }
);
```

**Why it matters for architecture:** vendor lock-in on feature flags is real — LD SDK calls are scattered throughout the codebase. OpenFeature lets you start with Unleash (open source, self-hosted, zero SaaS cost) and migrate to a commercial tool later without touching application code.

### Unleash (open source, self-hosted)

[Unleash](https://www.getunleash.io/) is the leading open-source option — a self-hosted toggle server with Node.js/Java/Python SDKs and a web UI. Evaluation is local (cache synced from server), so latency is the same as commercial tools.

```typescript
import { initialize } from 'unleash-client';

const unleash = initialize({
  url: 'https://unleash.internal/api/',
  appName: 'checkout-service',
  customHeaders: { Authorization: process.env.UNLEASH_API_TOKEN },
});

// isEnabled is synchronous after initialization — local cache
if (unleash.isEnabled('new-checkout-flow', { userId: user.id })) {
  return newCheckoutHandler(req, res);
}
```

Chosen when regulatory requirements prevent SaaS, or when teams want full control of the toggle infrastructure.

### Targeting rules

All mature platforms support rules that evaluate flag state against context attributes:

- **User targeting** — `userId` equals specific values (QA users, internal staff)
- **Attribute targeting** — `plan == 'enterprise'` OR `country in ['GB', 'DE']`
- **Percentage rollout** — deterministic hash of `userId` → consistent bucket (same user always gets same variant across requests and sessions)
- **Date/time rules** — flag auto-enables after a scheduled date
- **Segment reuse** — define "beta users" segment once, reference across many flags

**Sticky evaluation** is critical for gradual rollouts: the same user must get the same variant on every request. Hashing the user ID against the rollout percentage achieves this deterministically — no server-side state needed.

### Flag lifecycle & flag debt

Release flags are **temporary**. The intended lifecycle:

```
Create  →  Deploy dark  →  Enable for QA  →  Ramp 1% → 10% → 100%  →  Remove flag + dead code
```

The last step is the one teams skip. **Flag debt** accumulates when:
- Code has `if (flag) { newPath } else { oldPath }` where the old path has been dead for months
- Evaluation calls that always return `true` add latency and cognitive overhead for no benefit
- No one knows which flags are safe to delete

**Hygiene practices:**
- Assign an **owner** and a **TTL** (e.g., 30 or 90 days) to every release flag at creation
- Alert when a flag exceeds its TTL with no cleanup PR
- Use separate namespaces/tags for permanent ops flags vs temporary release flags
- Some teams run CI checks that fail if a release flag has been 100%-on for > 90 days without a cleanup ticket

**Interview framing:** "How do you safely roll out a risky feature?" — Deploy behind a flag, ramp from internal → 1% → 10% → 100% while watching error rate and MTTR, with an instant kill-switch. That arc (progressive delivery + observability + kill-switch) is exactly the DORA elite-performer playbook. Follow-up: "What's the operational cost?" — flag debt if you don't have a cleanup discipline; evaluate that honestly.

## Engineering Excellence Priority Summary

| Topic | Priority |
| --- | --- |
| **Craftsmanship & Review** | |
| Code review (small PRs, what to look for) | **Critical** |
| Clean code / naming / small functions | **Important** |
| Definition of Done & quality gates | **Important** |
| Technical-debt quadrant & paydown | **Deep** |
| **Testing Strategy** | |
| Test pyramid / trophy (cost vs confidence) | **Critical** |
| Test types: smoke, regression, acceptance, exploratory | **Important** |
| TDD (red-green-refactor) | **Learn** |
| BDD / Cucumber (Given-When-Then, Gherkin) | **Know** |
| Contract testing (Pact, Postman/Newman) | **Know** |
| Coverage as signal not target | **Important** |
| Shift-left | **Important** |
| Performance testing (k6 — load/stress/soak) | **Important** |
| Penetration testing (OWASP ZAP, DAST) | **Know** |
| **Delivery Metrics** | |
| DORA four keys | **Deep** |
| Trunk-based dev + feature flags | **Important** |
| Canary / blue-green | **Important** |
| **Feature Flags** | |
| Flag types (release, rollout, kill-switch, ops) | **Deep** |
| LaunchDarkly / OpenFeature / Unleash | **Important** |
| Targeting rules + sticky evaluation | **Important** |
| Flag lifecycle & flag debt hygiene | **Important** |
| **Reliability** | |
| Incident lifecycle (mitigate first) | **Deep** |
| Blameless postmortems / 5 Whys | **Important** |
| Error budgets | **Important** (see §26.2) |
| **Behavioral Interviews** | |
| STAR framework | **Critical** |
| Must-have story themes (8 themes) | **Critical** |
| Senior leveling signals | **Deep** |
| Questions to ask interviewers | **Important** |

---


# 32. Agile & Delivery

Interviewers rarely ask "what is Scrum?" as a quiz — they ask it to find out whether you've *worked* inside a delivery process and formed opinions about it. A senior engineer is expected to estimate credibly, slice work so it flows, run a useful refinement session, and know when a ceremony has become theater. The frameworks (Scrum, Kanban) are simple; the value is in the *judgment* — knowing that story points measure relative effort not hours, that a daily standup is for surfacing blockers not status-reporting to a manager, and that velocity is a planning tool, never a productivity score to compare across teams. This section covers the frameworks and ceremonies from the engineer's seat, plus the estimation practices that trip people up.

## 32.1 Agile Foundations

Agile is a set of *values*, not a process. The **Agile Manifesto** (2001) states four preferences — note it values the right side too, just *less*:

```text
Individuals and interactions   over   processes and tools
Working software               over   comprehensive documentation
Customer collaboration         over   contract negotiation
Responding to change           over   following a plan
```

The core bet versus **waterfall** (sequential: requirements → design → build → test → release) is that requirements are *discovered*, not known upfront, so you work in short **iterations** that produce working software, get feedback, and adapt. Scrum and Kanban are two concrete frameworks that implement these values differently.

## 32.2 Scrum

The most common framework: time-boxed **sprints** (usually 1–2 weeks) that each produce a potentially shippable increment.

**Roles:**

- **Product Owner (PO)** — owns and prioritizes the *backlog*; decides *what* and *why*.
- **Scrum Master (SM)** — facilitates the process, removes blockers, shields the team; *not* a manager.
- **Developers** — the cross-functional team that decides *how* and *how much* fits in a sprint.

**Artifacts:**

- **Product backlog** — the prioritized master list of everything that might be built.
- **Sprint backlog** — the slice the team commits to this sprint.
- **Increment** — the working output, meeting the Definition of Done (§31.1).

**Events (ceremonies):**

| Event | When | Purpose |
| --- | --- | --- |
| **Sprint Planning** | Start of sprint | Pull from backlog into sprint; agree on a goal |
| **Daily Standup** | Daily, ~15 min | Surface progress & **blockers** (not status for a boss) |
| **Sprint Review** | End of sprint | Demo the increment to stakeholders; get feedback |
| **Retrospective** | End of sprint | Improve the *process*: what to keep/stop/start |

The **retro** is the engine of continuous improvement and the one engineers most often undervalue — it's where the team fixes *how it works*, not just *what it builds*.

## 32.3 Kanban

Kanban optimizes for **continuous flow** rather than fixed time-boxes. Work moves across a board (`To Do → In Progress → Review → Done`) and the central discipline is the **WIP (work-in-progress) limit** — a cap on how many items can sit in a column at once.

**Why WIP limits matter:** limiting WIP forces the team to *finish* before *starting*, exposes bottlenecks (the column that fills up is the constraint), and reduces context-switching. It's a direct application of queuing theory — less WIP means shorter cycle time.

- **Lead time** — from request created → delivered (the customer's clock).
- **Cycle time** — from work *started* → delivered (the team's clock).

**Scrum vs Kanban:**

| | Scrum | Kanban |
| --- | --- | --- |
| Cadence | Fixed sprints | Continuous flow |
| Commitment | Sprint backlog | No sprint commitment |
| Roles | PO, SM, devs | No prescribed roles |
| Change mid-cycle | Discouraged | Anytime |
| Best for | Feature teams, predictable cadence | Support/ops, variable priorities |

**Scrumban** blends them — Kanban flow with some Scrum ceremonies — and is common in practice.

## 32.4 Refinement, Grooming & Planning

These terms get conflated in interviews; knowing the distinction signals experience.

- **Backlog refinement** (a.k.a. **grooming** — the older term, now deprecated for its connotations) — an *ongoing* activity where the team clarifies upcoming backlog items, adds detail and acceptance criteria, splits large items, and estimates. It keeps the top of the backlog "ready."
- **Sprint planning** — a *point-in-time* event at sprint start where the team selects refined items into the sprint and commits to a goal.

The bridge between them is two checklists:

- **Definition of Ready (DoR)** — when is a story ready to be pulled into a sprint? (Clear acceptance criteria, estimated, no blocking dependencies, small enough.)
- **Definition of Done (DoD)** — when is it actually finished? (See §31.1.)

**Good user stories follow INVEST:** **I**ndependent, **N**egotiable, **V**aluable, **E**stimable, **S**mall, **T**estable. The classic template — *"As a `<role>`, I want `<goal>` so that `<benefit>`"* — plus explicit **acceptance criteria** (often Given/When/Then) that double as test cases.

## 32.5 Estimation & Story Points

The most misunderstood topic. **Story points estimate relative *effort/complexity/uncertainty*, not hours.** The reasons this matters:

- Humans are bad at absolute time estimates but decent at *relative* sizing ("this is about twice as hard as that").
- Points decouple estimation from individual speed — a senior and a junior can agree a task is "5 points" even if it takes them different wall-clock time.
- Converting points to hours destroys the benefit and invites the dysfunction of treating estimates as commitments.

**How it works in practice:**

- **Scale** — usually a **Fibonacci-like** sequence (1, 2, 3, 5, 8, 13, 21). The widening gaps reflect that larger items carry exponentially more uncertainty; an "8" really means "we're not sure — consider splitting."
- **Planning poker** — everyone estimates simultaneously (cards/app) to avoid anchoring on the loudest voice; divergent numbers trigger a discussion that surfaces hidden assumptions. *That conversation is the real value*, more than the number.
- **T-shirt sizing** (S/M/L/XL) — coarser, good for early/epic-level estimation before stories are refined.
- **Velocity** — the average points completed per sprint, used to forecast *how much fits next sprint*. Critical caveat: velocity is a **per-team planning tool, never a productivity metric** and never comparable across teams (a team can inflate points trivially). Weaponizing velocity is a classic anti-pattern.

A pragmatic counter-movement, **#NoEstimates**, argues for slicing work into uniformly small pieces and just counting throughput — worth knowing as a talking point.

**Interview framing:** "How do you estimate?" — Explain points as *relative effort under uncertainty*, describe planning poker's value as *surfacing disagreement*, and explicitly call out that you'd resist converting points to hours or comparing velocity across teams. Mention that the act of breaking work down for estimation is itself a design exercise that catches unknowns early.

## Agile & Delivery Priority Summary

| Topic | Priority |
| --- | --- |
| Scrum roles / artifacts / events | **Important** |
| Standup = blockers, retro = improvement | **Important** |
| Kanban + WIP limits + flow | **Important** |
| Lead time vs cycle time | **Learn** |
| Scrum vs Kanban trade-offs | **Learn** |
| Refinement vs planning; DoR vs DoD | **Important** |
| INVEST + acceptance criteria | **Learn** |
| Story points = relative effort (not hours) | **Critical** |
| Planning poker, Fibonacci, velocity caveats | **Important** |
| Agile Manifesto values | **Know** |

---


# 33. Documentation

Senior engineers are judged not just on the code they write but on the *decisions* they leave behind in a form the next person can follow. Documentation is an engineering deliverable, not an afterthought — and the skill is matching the *type* of document to its *audience and lifespan*. A README onboards a newcomer; an ADR preserves *why* a decision was made long after everyone who made it has left; a runbook lets a half-asleep on-call engineer recover a service at 3 a.m. The anti-patterns are equally important to recognize: documentation that drifts out of sync with reality is worse than none, which is why the modern discipline is **"docs as code"** — docs live in version control, are reviewed in PRs, and (where possible) are generated or tested. This section covers the document types and the two interviewers care about most: ADRs and design docs.

## 33.1 Documentation Types & Audiences

Different documents serve different readers and decay at different rates:

| Type | Audience | Lifespan | Answers |
| --- | --- | --- | --- |
| **README** | New contributors | Living | "How do I run/use this?" |
| **ADR** | Future engineers | Immutable record | "*Why* did we decide this?" |
| **RFC / design doc** | Reviewers, pre-build | Frozen after decision | "*Should* we build this, and how?" |
| **Runbook** | On-call / ops | Living | "How do I operate/fix this?" |
| **API docs** (OpenAPI) | API consumers | Generated | "What endpoints/shapes exist?" |
| **Inline comments** | Code maintainers | With the code | "*Why* is this line non-obvious?" |

**Docs as code** — keep docs in the repo (Markdown), review them in the same PR as the change, and prefer *generated* docs (OpenAPI from code/annotations, typedoc) over hand-maintained ones that drift. The litmus test: *if someone changes the behavior, will the doc break loudly or rot silently?* Prefer the former.

**Comments vs docs** (ties to §31.1): comments explain *why* a specific line is the way it is; docs explain the system. Don't narrate *what* the code does — name things well instead.

## 33.2 ADRs (Architecture Decision Records)

An **ADR** is a short, immutable record of a single significant architectural decision and its context. Popularized by Michael Nygard; the lightweight **MADR** template is common. The point is to capture the *reasoning* — the alternatives and trade-offs — so future engineers don't re-litigate settled decisions or, worse, reverse them without knowing why.

```markdown
# ADR 0007: Use PostgreSQL instead of MongoDB for the orders service

## Status
Accepted   (others: Proposed | Deprecated | Superseded by ADR-0012)

## Context
Orders are highly relational (line items, payments, refunds) and we need
ACID transactions across them. The team also has deep SQL experience.

## Decision
We will use PostgreSQL as the primary datastore for the orders service.

## Consequences
+ Strong consistency and transactional integrity for financial data.
+ Mature tooling, team familiarity.
- Less flexible for rapidly-changing document shapes (acceptable here).
- Requires schema migrations discipline (we adopt Flyway).
```

**Rules that make ADRs work:**

- **One decision per ADR**, numbered sequentially, stored *in the repo* (`/docs/adr`).
- **Immutable** — you don't edit an accepted ADR; you write a new one that **supersedes** it and update the old one's status. This preserves the historical record of *why we thought X, then changed to Y*.
- **Write them when the decision is hard to reverse or expensive to change** — datastore choice, sync vs async, framework, API style. Not for routine choices.

**Interview framing:** "How do you document architecture decisions?" — Describe ADRs, emphasize capturing *context and rejected alternatives* (the "why not"), and the immutability/supersede model. This signals you think about the engineers who come after you. ADRs are the durable output of the decision process in §36.3.

## 33.3 RFCs / Design Docs

An **RFC** ("request for comments") or **design doc** is written *before* building something non-trivial, to align reviewers and surface problems while they're cheap. Where an ADR records a decision *after* it's made, an RFC *proposes* and *seeks feedback*.

```text
Typical design-doc structure:
  - Title, author, status, reviewers
  - Context / problem statement
  - Goals & non-goals          ← scoping; prevents creep
  - Proposed design            ← diagrams, data model, APIs
  - Alternatives considered    ← shows you did the homework
  - Trade-offs & risks
  - Rollout / migration plan
  - Open questions
```

The review (comments on the doc) is the deliverable as much as the doc — it's where senior reviewers catch design flaws before code exists. **RFC vs ADR:** an RFC is the *forward-looking proposal and discussion*; the ADR is the *terse, durable record of what was decided*. A big RFC often spawns several ADRs.

## 33.4 Runbooks & Operational Docs

A **runbook** (or playbook) is operational documentation for running and recovering a service — written for the on-call engineer under stress (ties to §31.4 and §26). A good alert *links directly to its runbook*.

```text
Runbook: "API 5xx rate > 2%"
  Symptoms:   elevated 5xx in dashboard X, customer reports
  Likely causes: DB connection pool exhaustion, downstream timeout
  Diagnosis:  check pool metrics (link), check dependency health (link)
  Mitigation: 1) scale out pods   2) flip feature flag Y off
  Escalation: page DB on-call if pool is healthy but errors persist
```

The goal: a stressed engineer at 3 a.m. can follow it without tribal knowledge. Keep them living — update them after every incident's postmortem.

## 33.5 Diagrams as Code

Diagrams communicate architecture faster than prose, and keeping them *as code* (text → rendered) means they live in version control and stay in sync.

- **C4 model** — four zoom levels: **Context** (system + users + external systems) → **Container** (apps/services/datastores) → **Component** (inside a container) → **Code** (rarely needed). The discipline is picking the *right level* for the audience — execs want Context, engineers want Container/Component.
- **Mermaid** — Markdown-native diagrams (flowcharts, **sequence diagrams**, ER, state) that render in GitHub/GitLab/docs directly. The pragmatic default.
- **PlantUML** — more powerful/older, renders a wider range; needs a renderer.

**Sequence diagrams** are especially interview-relevant for explaining request flows (auth handshakes, distributed transactions). Being able to sketch one quickly is a senior signal.

## Documentation Priority Summary

| Topic | Priority |
| --- | --- |
| ADRs (context, immutability, supersede) | **Important** |
| Doc types ↔ audience/lifespan | **Important** |
| Docs as code | **Learn** |
| RFC / design doc structure | **Important** |
| RFC vs ADR | **Learn** |
| Runbooks (alert-linked, operational) | **Learn** |
| C4 model | **Learn** |
| Mermaid / sequence diagrams | **Know** |

---

---

_End of Part 7. Continue to **Part 8** (Solution Architecture) in [`08_SolutionArchitecture_34-36.md`](./08_SolutionArchitecture_34-36.md), or return to the [README](./README.md)._
