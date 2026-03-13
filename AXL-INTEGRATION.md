# OMO ↔ AXL Integration Plan

## What Is OMO?

**Oh My OpenAgent** (OMO) is an agent harness — a plugin for OpenCode that transforms a single coding agent into a full AI development team. It's not a model, not a framework. It's the **orchestration layer** that sits between models and code.

### The Architecture (what makes it special)

```
┌─────────────────────────────────────────────────────────┐
│                    OMO Plugin Layer                       │
│                                                           │
│  ┌──────────────────────────────────────────────────┐    │
│  │ AGENTS (11)                                       │    │
│  │  Sisyphus → Orchestrator (Claude/Kimi/GLM)        │    │
│  │  Hephaestus → Deep Worker (GPT-5.3 Codex)        │    │
│  │  Prometheus → Strategic Planner (Claude/Kimi)      │    │
│  │  Atlas → Conductor for plan execution              │    │
│  │  Oracle → Architecture advisor (GPT-5.4)           │    │
│  │  Explore → Codebase grep (Grok Code)               │    │
│  │  Librarian → Docs/OSS search (Gemini Flash)        │    │
│  │  Metis → Consultant for planning                   │    │
│  │  Momus → Plan reviewer/critic (GPT-5.4)            │    │
│  │  Sisyphus-Junior → Task executor (Sonnet)          │    │
│  │  Multimodal-Looker → Visual inspection             │    │
│  └──────────────────────────────────────────────────┘    │
│                                                           │
│  ┌──────────────────────────────────────────────────┐    │
│  │ KEY INNOVATIONS                                    │    │
│  │  • Hashline Edit: content-hash anchored editing    │    │
│  │    (6.7% → 68.3% success on Grok Code Fast)       │    │
│  │  • Ralph Loop: autonomous persistence loop         │    │
│  │  • Todo Enforcer: yanks idle agents back to work   │    │
│  │  • Category Routing: auto-picks model per task     │    │
│  │  • IntentGate: analyzes true user intent           │    │
│  │  • Background Agents: 5+ specialists in parallel   │    │
│  │  • Skill-Embedded MCPs: context-scoped tools       │    │
│  │  • Comment Checker: no AI slop in comments         │    │
│  └──────────────────────────────────────────────────┘    │
│                                                           │
│  ┌──────────────────────────────────────────────────┐    │
│  │ 46 HOOKS (lifecycle interceptors)                  │    │
│  │  Session: recovery, compaction, context injection  │    │
│  │  Tool: hashline edit/read, file guard, label trunc │    │
│  │  Agent: model fallback, permission, babysitter     │    │
│  │  Loop: ralph-loop, todo-enforcer, stop-guard       │    │
│  └──────────────────────────────────────────────────┘    │
│                                                           │
│  ┌──────────────────────────────────────────────────┐    │
│  │ 26 TOOLS                                           │    │
│  │  hashline-edit, delegate-task, background-task,    │    │
│  │  ast-grep, lsp (rename/goto/refs/diagnostics),     │    │
│  │  interactive-bash, glob, grep, look-at,            │    │
│  │  skill-mcp, session-manager, task, call-omo-agent  │    │
│  └──────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### The Key Insight

OMO's core thesis: **agent failures are usually harness failures, not model failures.** The edit tool loses track of line numbers. The context window overflows. The agent forgets its task. OMO fixes these at the infrastructure level:

| Problem | OMO Solution |
|---------|-------------|
| Agent edits wrong lines (stale content) | **Hashline Edit** — content-hash validates every change |
| Agent stops halfway | **Ralph Loop + Todo Enforcer** — keeps going until truly done |
| Agent picks wrong model for subtask | **Category Routing** — `visual-engineering` → frontend model, `deep` → codex |
| Agent context window overflows | **Background Agents** — fire specialists, get results, context stays lean |
| Agent doesn't understand codebase | **Explore + Librarian** — dedicated agents for codebase/docs search |
| Agent writes sloppy comments | **Comment Checker** hook strips AI slop |
| Agent scope creeps | **Prometheus Planner** — interview → plan → execute (separation of concerns) |

---

## What AXL Already Does

AXL is a **design-to-code automation platform**:

```
Google Stitch Design → Code Generation → Preview Deploy → UX Review → Feedback Loop
                        (OMO agents)      (Railway)      (screenshot   (structured
                                                          comparison)   feedback)
```

AXL already spawns OMO agents via `CodingAgentRunner` — but only uses a **fraction** of OMO's capabilities. The current integration is essentially:

```python
# Current: spawn OMO as a subprocess with a context prompt
result = await self._spawn_agent(context_prompt=context_prompt)
```

This is like having a Formula 1 car and only using first gear.

---

## Integration Opportunities

### 1. CATEGORY-AWARE TASK ROUTING (High Impact, Low Effort)

**Current state:** AXL spawns a generic OMO agent for every coding task.

**What OMO offers:** Category-based model routing. When Sisyphus delegates, it picks a *category* not a model:

| Category | Best For | Model |
|----------|----------|-------|
| `visual-engineering` | UI/UX, design matching, component styling | Gemini 3.1 Pro |
| `deep` | Autonomous research + full implementation | GPT-5.3 Codex |
| `quick` | Single-file changes, typos, config tweaks | Fast cheap model |
| `ultrabrain` | Architecture decisions, hard logic | GPT-5.4 xhigh |

**Integration:** AXL's `CodingAgentRunner` should pass category hints based on the flow type:
- Login/checkout UI flows → `visual-engineering`
- API endpoint implementation → `deep`
- Config/env changes → `quick`
- Schema design → `ultrabrain`

```python
# Proposed: category-aware dispatch
category = self._classify_flow(flow_key, design_spec)
result = await self._spawn_agent(
    context_prompt=context_prompt,
    category=category  # OMO routes to optimal model
)
```

**Expected improvement:** Better model-task fit → fewer iterations to convergence.

---

### 2. PROMETHEUS PLANNING BEFORE CODING (High Impact, Medium Effort)

**Current state:** AXL builds a context prompt and hands it directly to the coding agent.

**What OMO offers:** Prometheus — a strategic planner that interviews, identifies scope, and builds a verified plan *before* any code is touched. Momus reviews the plan for correctness.

**Integration:**
```
Current:  Design → Context Builder → Coding Agent → Deploy → Review
Proposed: Design → Context Builder → Prometheus Plan → Momus Review → Atlas Execution → Deploy → Review
```

For each loop iteration:
1. Prometheus reads the design spec + previous feedback
2. Generates a `.sisyphus/plans/iteration-N.md` with atomic steps
3. Momus (GPT-5.4) reviews the plan for completeness
4. Atlas orchestrates execution of the plan steps
5. Each step delegated to the right specialist (visual, deep, quick)

**Expected improvement:** Fewer wasted iterations. Plans catch ambiguity *before* code is written.

---

### 3. HASHLINE EDIT FOR REVIEW FEEDBACK (High Impact, Low Effort)

**Current state:** AXL's review agents produce feedback as text. The coding agent then has to figure out which lines to change.

**What OMO offers:** Hashline — every line has a content-hash anchor. The review agent can reference specific lines by hash, and the coding agent can edit them without reproduction errors.

**Integration:** When the UX review agent generates findings, include hashline references:
```
Finding: Button color doesn't match design
File: src/routes/login/+page.svelte
Line: 47#XJ  → background-color: #3b82f6  (should be design token --primary)
```

The coding agent receives hash-anchored feedback → edits are precise, not fuzzy.

**Expected improvement:** 6.7% → 68.3% edit success rate (their benchmark). Massive reduction in "wrong line edited" bugs.

---

### 4. RALPH LOOP FOR CONVERGENCE (High Impact, Medium Effort)

**Current state:** AXL has a fixed iteration count (default 10). Each iteration is a discrete agent invocation.

**What OMO offers:** Ralph Loop — a self-referential autonomous loop that doesn't stop until the task is 100% done. With `--completion-promise` it verifies via Oracle before claiming done.

**Integration:** Instead of fixed iterations, use Ralph Loop with convergence as the exit condition:
```
/ulw-loop "Implement flow {flow_key} matching design spec. Do not stop until:
1. All visual findings are addressed
2. Lint gate passes (ESLint + Ruff)
3. Design comparison score > {threshold}"
--completion-promise="All UX review findings resolved, lint clean, convergence threshold met"
--max-iterations=20
```

The agent keeps iterating *within a single invocation* until it achieves convergence, rather than needing AXL to orchestrate discrete iterations externally.

**Expected improvement:** Faster convergence. The agent learns from its own mistakes within the same context window, rather than starting fresh each iteration.

---

### 5. BACKGROUND AGENTS FOR PARALLEL FLOW IMPLEMENTATION (High Impact, Medium Effort)

**Current state:** AXL processes flows sequentially (or spawns separate processes).

**What OMO offers:** Background agents — fire 5+ specialists in parallel, get results when ready. Context stays lean.

**Integration:** For a design with 5 flows (login, dashboard, settings, checkout, profile):
```
Sisyphus spawns:
  background_task(category="visual-engineering", prompt="Implement login flow...")
  background_task(category="visual-engineering", prompt="Implement dashboard flow...")
  background_task(category="visual-engineering", prompt="Implement settings flow...")
  background_task(category="visual-engineering", prompt="Implement checkout flow...")
  background_task(category="visual-engineering", prompt="Implement profile flow...")
```

All 5 run in parallel using git worktrees. Results merge when all complete.

**Expected improvement:** 5x throughput for multi-flow designs.

---

### 6. DESIGN SYSTEM ENFORCEMENT VIA VISUAL CATEGORY (Medium Impact, Low Effort)

**Current state:** AXL's coding agents sometimes produce inconsistent styling across iterations.

**What OMO offers:** The `visual-engineering` category has a **mandatory 4-phase workflow**:
1. Analyze existing design system (colors, spacing, typography)
2. If none exists → build one first
3. Build WITH the system (never hardcoded values)
4. Verify consistency before claiming done

**Integration:** Add a discipline rule at `.sisyphus/rules/design-system.md` in the generated project:
```markdown
Every visual change MUST use design tokens from the extracted Stitch design spec.
Hardcoded colors, spacing, or typography are REJECTED.
```

**Expected improvement:** Design consistency across flows and iterations.

---

### 7. FORK DETECTION + CONFLICT RESOLUTION (Medium Impact, Medium Effort)

**Current state:** AXL has `ForkDetector` and `ConflictManager` but they operate at the orchestrator level.

**What OMO offers:** Git worktree management built into the background agent system. Each agent works in an isolated worktree. Conflicts detected at merge time.

**Integration:** Use OMO's worktree system for agent isolation:
```
Agent A (login flow) → worktree: .worktrees/flow-login
Agent B (dashboard) → worktree: .worktrees/flow-dashboard
Merge back to iteration branch when both complete
```

AXL's `ConflictManager` handles merge conflicts, but OMO's worktree management prevents file-level collisions during execution.

---

### 8. INIT-DEEP FOR AUTO-GENERATED AGENTS.MD (Low Effort, Medium Impact)

**Current state:** Generated projects have no agent context files.

**What OMO offers:** `/init-deep` auto-generates hierarchical `AGENTS.md` files throughout the project tree.

**Integration:** After initial code generation, run `/init-deep` to create context files. Every subsequent iteration benefits from agents understanding the project structure immediately (no exploration overhead).

---

## What's Missing in AXL (Gaps to Fill)

### GAP 1: No LSP Integration
AXL's lint gate runs ESLint + Ruff as batch processes. OMO has **real-time LSP** — rename, goto-definition, find-references, diagnostics. The coding agent could use LSP to:
- Rename symbols safely across files
- Find all references before changing an interface
- Get inline diagnostics without running full lint

**To build:** Wire OMO's LSP tools into the coding agent's available toolset.

### GAP 2: No AST-Grep
AXL's agents search code with plain grep. OMO has **AST-aware search** across 25 languages. This means:
- Find all React components that accept a `color` prop
- Find all Svelte `{#each}` blocks without keys
- Rewrite patterns structurally, not textually

**To build:** Enable AST-Grep tool in the coding agent configuration.

### GAP 3: No MCP Integration for Design Context
OMO has built-in MCPs for web search (Exa), docs (Context7), and code search (Grep.app). AXL could add a **Stitch MCP** — an on-demand tool that lets the coding agent query the design spec directly:
```
mcp_call("stitch", "get_flow_spec", { flow: "login" })
mcp_call("stitch", "get_color_palette", {})
mcp_call("stitch", "get_component_variants", { component: "Button" })
```

**To build:** Implement a Stitch MCP server that wraps AXL's design ingestion API.

### GAP 4: No Human Steering Integration
AXL has nudges (`NudgeStore`) but they're injected at the orchestrator level. OMO has **interactive bash sessions via tmux** — a human can observe and steer the agent in real time.

**To build:** Expose the OMO tmux session in AXL's dashboard. Users can watch agents work and inject corrections mid-iteration.

### GAP 5: No Agent Telemetry Dashboard
AXL tracks loop telemetry (started, completed, failed). OMO's agents emit richer data — tool calls, delegate chains, background task status, todo progress.

**To build:** Stream OMO's plugin events to AXL's telemetry system. Show agent decision trees in the dashboard.

### GAP 6: No Multi-Model Cost Optimization
AXL spawns one model type. OMO routes to the cheapest capable model per task category. A simple color fix shouldn't use Opus — it should use Flash.

**To build:** Pass AXL's budget constraints to OMO's category config. Set cost ceilings per category.

### GAP 7: Comment Quality Enforcement
AXL's generated code sometimes has AI-quality comments ("This function handles the login flow" on a function called `handleLoginFlow`). OMO's Comment Checker hook strips these.

**To build:** Enable the comment-checker hook in the coding agent configuration.

### GAP 8: Session Recovery
If an agent crashes mid-iteration, AXL retries from scratch. OMO has **session recovery** — resume from where the agent left off, preserving context and progress.

**To build:** Use OMO's session recovery hooks instead of AXL's full-retry logic.

---

## Implementation Roadmap

### Phase 1: Quick Integration (1-2 days)
- [ ] Enable category-aware routing in `CodingAgentRunner`
- [ ] Enable hashline edit for review feedback
- [ ] Enable comment-checker hook
- [ ] Run `/init-deep` on generated projects
- [ ] Add design system discipline rule

### Phase 2: Deep Integration (1 week)
- [ ] Replace fixed iteration count with Ralph Loop
- [ ] Implement Prometheus planning before coding
- [ ] Enable background agents for parallel flow implementation
- [ ] Wire LSP + AST-Grep tools into agent config
- [ ] Implement session recovery

### Phase 3: Custom Extensions (2 weeks)
- [ ] Build Stitch MCP server (design spec queries)
- [ ] Stream OMO events to AXL telemetry dashboard
- [ ] Expose tmux sessions in AXL dashboard for human steering
- [ ] Implement multi-model cost optimization
- [ ] Build Momus review gate for plans

### Phase 4: Autonomous Convergence (ongoing)
- [ ] AutoJe integration — continuous overnight improvement of generated code
- [ ] Cross-project learning — agents share patterns across generated projects
- [ ] Self-improving prompts — Prometheus evolves its own planning templates
