# Architectural Recommendation: Polecat E2E Verification Timeout

> **Bead**: hq-47tea | **Priority**: P1 | **Date**: 2026-02-16
> **Author**: EnterpriseArch/polecats/furiosa

---

## 1. Root Cause Analysis

### The Problem

Polecats consistently exhaust their session budget before completing the commit+submit cycle when E2E tests are part of the verification pipeline. The CygnusWealthApp verification pipeline takes 9-11 minutes:

| Step | Duration | Notes |
|---|---|---|
| `npm install` | ~2 min | Full dependency resolution |
| Playwright browser install | ~3 min | Downloads chromium + firefox binaries |
| Build | ~1 min | Vite production build |
| Unit tests | ~1 min | Fast, not the bottleneck |
| E2E tests (64 tests, 2 browsers) | ~4-5 min | Playwright across chromium + firefox |
| **Total verification** | **~9-11 min** | Before commit is even attempted |

### Why This Kills Polecats

Polecats are ephemeral, unattended agents with bounded session budgets. Their workflow is: **investigate -> implement -> verify -> commit -> gt done**. The verification step is a blocking synchronous gate — the polecat must wait for the full pipeline to complete before it can commit.

The fundamental mismatch: polecats need to spend their budget on **cognitive work** (reading code, understanding the problem, implementing the fix) but the E2E pipeline consumes the budget on **idle waiting** (watching npm install, browser download, test execution). By the time E2E finishes, there is no budget left to commit.

### Evidence

- **hq-3asww** (CygnusWealthApp): Required zero code changes. Dependencies were already correct. E2E was running at 2m48s into a 5m timeout when the session died. The polecat did its job perfectly but was killed waiting for verification.
- **hq-04rc8** (CygnusWealthApp): Changed 2 lines in `package.json`. Build passed, unit tests were running when the session died. Never reached E2E, let alone commit.

### Structural Root Cause

The polecat verification model treats all tests as equal pre-commit gates. This was appropriate when verification meant "run unit tests" (30 seconds). It breaks when verification means "install browsers, build the app, run 64 browser tests across 2 engines" (11 minutes). The model conflates **correctness confidence** (did the polecat's change break anything?) with **comprehensive verification** (does the entire E2E suite still pass?).

---

## 2. Solution Evaluation

### Option 1: Split Verification — Commit After Unit Tests, Verify E2E Separately

**Concept**: Polecats run unit tests and type-checking as the pre-commit gate. E2E verification is delegated to a separate downstream step (refinery verification pass or dedicated verification agent).

| Criterion | Assessment |
|---|---|
| Solves the timeout? | **Yes** — polecats complete in 3-4 minutes instead of 11+ |
| Works with ephemeral model? | **Yes** — no change to polecat lifecycle |
| Risk of broken commits? | **Moderate** — unit tests catch most regressions; E2E failures are caught before merge |
| Implementation complexity | **Medium** — requires a verification step in the refinery pipeline |
| Aligns with enterprise E2E strategy? | **Yes** — the strategy already separates E2E from unit tests in CI triggers |

**Verdict: RECOMMENDED (primary)**

### Option 2: Pre-install Playwright Browsers in Worktree Templates

**Concept**: Bake Playwright browser binaries into the worktree template so polecats skip the 3-minute browser download.

| Criterion | Assessment |
|---|---|
| Solves the timeout? | **Partially** — saves ~3 minutes, but 6-8 min remains |
| Works with ephemeral model? | **Yes** — worktree templates are a known concept |
| Risk of broken commits? | **None** — purely an optimization |
| Implementation complexity | **Low** — one-time template setup + maintenance |
| Aligns with enterprise E2E strategy? | **Neutral** — infrastructure optimization |

**Verdict: RECOMMENDED (complementary) — but insufficient alone**

### Option 3: Increase Polecat Session/Context Budget

**Concept**: Give polecats more time/tokens to accommodate long-running E2E pipelines.

| Criterion | Assessment |
|---|---|
| Solves the timeout? | **Maybe** — depends on whether budget is time-based or token-based |
| Works with ephemeral model? | **Poorly** — longer sessions mean more cost and slower feedback |
| Risk of broken commits? | **None** |
| Implementation complexity | **Low** — configuration change |
| Aligns with enterprise E2E strategy? | **No** — the strategy says E2E should not block individual commits |

**Verdict: NOT RECOMMENDED** — This treats the symptom, not the disease. If the budget is context-window-based, idle waiting still consumes it. If time-based, 15-20 minute sessions make polecats expensive and slow. The problem is that polecats shouldn't be waiting for E2E at all, not that they need more time to wait.

### Option 4: Post-Merge CI Gate for E2E

**Concept**: E2E tests run as a CI pipeline after merge to main, not as a polecat responsibility.

| Criterion | Assessment |
|---|---|
| Solves the timeout? | **Yes** — polecats never run E2E |
| Works with ephemeral model? | **Yes** — completely decouples polecats from E2E |
| Risk of broken commits? | **Higher** — broken code can merge; regression caught later |
| Implementation complexity | **Medium** — requires CI infrastructure (GitHub Actions already exists) |
| Aligns with enterprise E2E strategy? | **Yes** — strategy already defines "E2E (mocked): Every PR, merge to main" |

**Verdict: RECOMMENDED (for network-dependent E2E)** — The enterprise E2E strategy already mandates this for network-dependent tests. Extending it to all E2E as a CI gate (rather than polecat responsibility) is a natural evolution. However, this should complement split verification, not replace pre-merge E2E entirely.

### Option 5: Polecat Checkpointing

**Concept**: Polecats save their progress and a new session resumes where the previous one left off.

| Criterion | Assessment |
|---|---|
| Solves the timeout? | **Yes** — work continues across sessions |
| Works with ephemeral model? | **Contradicts it** — checkpointing makes polecats stateful |
| Risk of broken commits? | **None** |
| Implementation complexity | **Very High** — requires serializing agent state, worktree state, test progress |
| Aligns with enterprise E2E strategy? | **Neutral** |

**Verdict: NOT RECOMMENDED** — This fundamentally changes the polecat model from ephemeral to stateful. The complexity of serializing and resuming agent cognitive state is enormous. GT's strength is that polecats are disposable — checkpointing undermines that. The problem is better solved by not requiring polecats to do work that needs checkpointing.

### Option 6: Parallel Test Execution

**Concept**: Run E2E tests in parallel across workers to reduce wall-clock time.

| Criterion | Assessment |
|---|---|
| Solves the timeout? | **Partially** — E2E portion drops from ~5 min to ~2 min, but npm install + browser install still take 5 min |
| Works with ephemeral model? | **Yes** |
| Risk of broken commits? | **None** |
| Implementation complexity | **Low-Medium** — Playwright supports sharding natively |
| Aligns with enterprise E2E strategy? | **Yes** — strategy already notes "single worker in CI" as configurable |

**Verdict: RECOMMENDED (complementary)** — Useful optimization but insufficient alone. The 5-minute infrastructure overhead (npm + browsers) dominates regardless of test parallelism. Best applied in the CI/verification layer where it has the most impact.

---

## 3. Recommended Solution: Tiered Verification with Verification Handoff

### Architecture

Restructure the verification pipeline into two tiers, splitting responsibility between polecat (fast feedback) and refinery/CI (comprehensive verification):

**Tier 1 — Polecat Pre-Commit Gate (target: < 3 minutes)**
- TypeScript type-checking (`tsc --noEmit`)
- Unit tests (`npm test -- --run`)
- Lint (if configured)
- This is what the polecat must pass before committing

**Tier 2 — Verification Agent / CI Post-Commit Gate (target: 5-10 minutes)**
- Build verification (`npm run build`)
- E2E tests (`npm run test:e2e`)
- Cross-browser validation (Playwright chromium + firefox)
- This runs after the polecat commits but before merge to main

### Why This Works Within GT's Autonomous Agent Model

1. **Polecats remain ephemeral and fast** — they do cognitive work (investigate, fix, verify fast tests, commit) within their natural budget
2. **Verification is not lost** — it moves to the refinery pipeline or a dedicated verification step where time budgets are appropriate for long-running tests
3. **The feedback loop is preserved** — if E2E fails in Tier 2, a new bead is created for the regression and a new polecat is dispatched to fix it
4. **Already aligned with enterprise strategy** — the E2E testing strategy document already separates test types by trigger (unit: every push; E2E mocked: every PR; E2E network: nightly). This recommendation operationalizes that separation in the polecat model

### Complementary Optimizations

These should be implemented alongside the tiered verification model:

**A. Pre-warm Worktree Templates**
- Pre-install `node_modules` in worktree templates for rigs with heavy dependencies
- Pre-install Playwright browsers for CygnusWealthApp worktree template
- Reduces Tier 2 verification time from ~11 min to ~6 min

**B. Parallel E2E Execution in CI**
- Configure Playwright sharding in CI (2-4 shards)
- Run chromium and firefox in parallel rather than sequential
- Reduces E2E portion from ~5 min to ~2 min

**C. Dependency Caching**
- Leverage npm cache or `node_modules` caching between verification runs on the same worktree
- Skip `npm install` if `package-lock.json` hasn't changed

---

## 4. Implementation Plan

### Phase 1: Define Polecat Verification Tiers (Mayor)

**Component**: GT mayor/daemon configuration
**What changes**: Define a rig-level configuration that specifies which verification commands belong to Tier 1 (polecat pre-commit) vs Tier 2 (post-commit verification)
**Where**: Mayor rig configuration (e.g., `rigs.json` or per-rig config)
**Owner**: Mayor

### Phase 2: Update Polecat Verification Behavior (Polecat Rig)

**Component**: Polecat session workflow
**What changes**: Polecats check rig configuration for Tier 1 verification commands. After Tier 1 passes, commit and submit. Do not run Tier 2 commands.
**Where**: Polecat session prime/workflow scripts
**Owner**: Polecat rig maintainer

### Phase 3: Implement Verification Handoff (Refinery)

**Component**: Refinery pipeline
**What changes**: After a polecat commit is received, the refinery triggers Tier 2 verification. If Tier 2 fails, the commit is flagged and a regression bead is created.
**Where**: Refinery merge/verification workflow
**Owner**: Refinery maintainer

### Phase 4: Pre-warm CygnusWealthApp Worktree Template (Rig-specific)

**Component**: CygnusWealthApp worktree template
**What changes**: Include pre-installed `node_modules` and Playwright browser binaries in the worktree template. Add a maintenance script to refresh the template when `package-lock.json` changes.
**Where**: CygnusWealthApp rig configuration
**Owner**: CygnusWealthApp rig architect

### Phase 5: Enable Parallel E2E in CI (Rig-specific)

**Component**: CygnusWealthApp CI configuration
**What changes**: Configure Playwright sharding in `.github/workflows/ci.yml`. Split 64 tests across 2-4 parallel workers. Run chromium and firefox as separate CI jobs.
**Where**: CygnusWealthApp `.github/workflows/ci.yml`
**Owner**: CygnusWealthApp rig architect

### Dependency Order

```
Phase 1 (Mayor config)
  -> Phase 2 (Polecat behavior)
  -> Phase 3 (Refinery handoff)

Phase 4 (Worktree pre-warm) — independent, can proceed in parallel
Phase 5 (Parallel E2E) — independent, can proceed in parallel
```

### Expected Outcome

| Metric | Before | After |
|---|---|---|
| Polecat verification time | 9-11 min | 2-3 min |
| Polecat commit success rate | ~0% (for E2E rigs) | ~95%+ |
| E2E verification coverage | Attempted but never completed | 100% (in Tier 2) |
| Time to E2E feedback | Never (session dies) | 5-10 min post-commit |
| Regression detection | Not happening | Automated via Tier 2 -> bead creation |

---

## Summary

The core problem is a **responsibility misalignment**: polecats are ephemeral agents optimized for fast cognitive work, but E2E verification is a long-running infrastructure task. The solution is to split verification into tiers — polecats own fast verification (types + unit tests), and the refinery/CI pipeline owns comprehensive verification (build + E2E). This preserves GT's autonomous agent model while ensuring E2E coverage is not lost.
