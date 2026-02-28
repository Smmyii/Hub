# Hub Research — Multi-Agent Coordination Patterns

**Researched:** 2026-02-26
**Purpose:** Foundation research for Hub agentic workflow design

---

## Context Decay Prevention

**The core problem:** Every model tested gets worse as input context grows. This is not a minor optimization concern — it's fundamental to agent capability.

**Three mechanisms (Chroma Research, 18 models tested):**
- **Lost-in-the-Middle:** Models attend to beginning and end, poorly to the middle
- **Attention Scaling:** 10K tokens = 100M relationships; 100K tokens = 10B relationships
- **Distractor Interference:** Semantically similar but irrelevant content degrades beyond length alone

**What works (Anthropic — 39% performance boost, 84% token reduction):**
1. **Compaction** — summarize and reinitialize when approaching context limits. Start by clearing raw tool outputs, then compress broader history.
2. **Structured Note-Taking** — agents write notes to files outside the context window, read them back when needed. The agent writes for its *future self* which starts with blank context.
3. **Multi-Agent Architectures** — each subagent operates in its own clean context. Returns condensed summary (1-2K tokens) from 10K+ token explorations.

**Hub application:** Hub loads ~100 lines to orient. Bobs load ~100 lines. Deep context via JIT retrieval only.

---

## State Management (File-Based)

**Proven patterns:**
- JSON/YAML state files with metadata (workflow_id, current_step, timestamps)
- **Atomic writes** — write to temp file, rename over target. On POSIX, rename in same directory is atomic.
- **Checkpoint every meaningful step**, not just at completion
- **Ownership model** — only one agent writes to a given file at a time

**Framework approaches:**
- **LangGraph:** Checkpointer saves state at every super-step. PostgreSQL for production.
- **CrewAI:** Role-based pipeline, structured outputs between stages.
- **AutoGen:** Shared message pool, agents subscribe to relevant messages.
- **MetaGPT:** Shared message pool with SOPs. Assembly-line paradigm with standardized outputs.

**Anti-patterns:**
- Shared mutable state without coordination → race conditions
- No cleanup policy → "death by a thousand files"
- State files without version/sequence numbers → stale reads undetectable

---

## Agent Drift Prevention

**Scale of the problem:** Inter-agent misalignment accounts for ~40% of all observed breakdowns. Task Derailment (gradual deviation) is one of six inter-agent misalignment failure modes.

**What works:**
1. **Feature lists as guardrails** — structured JSON checklists where agents can only modify specific fields (Anthropic)
2. **Single feature per session** — avoids context exhaustion, prevents trying to do too much
3. **Judge agents** — secondary agent verifies critical outputs. "Often delivers the largest reliability improvement."
4. **Scaling rules in prompts** — simple queries get 1 agent, complex get 10+. Prevents over/under-investment.

**Hub application:** Work items have immutable objectives. Station gates validate before transitions. Bob does first-pass review before human.

---

## Context Window Management

**Tiered approach (LangChain):**
- **Working Memory:** Current conversation context
- **Short-term Memory:** Summarized recent interactions
- **Long-term Memory:** External storage accessed via retrieval

**Practical thresholds:**
- At 85% of context window → compress older tool calls
- Subagents return 1-2K token summaries from 10K+ explorations
- CLAUDE.md should not exceed ~150 lines
- Auto-memory loads first 200 lines of MEMORY.md

**Role-based filtering:** Each agent gets context for its role only. Research agent gets sources. Planning agent gets requirements. Execution agent gets the plan. No agent needs everything.

---

## Failure Modes

**Hard numbers:** 41-86% of multi-agent LLM systems fail in production. 79% of problems from specification and coordination, not technical implementation.

**14 failure modes (MASFT taxonomy):**

*Specification & Design (~35%):*
- Task Violation, Role Disobedience, Step Repetition, Context Loss, Termination Unawareness

*Inter-Agent Misalignment (~40%):*
- Conversation Reset, Clarification Failure, Task Derailment, Information Withholding, Input Disregard, Reasoning-Action Mismatch

*Task Verification (~25%):*
- Premature Termination, Incomplete Verification, Incorrect Verification

**The "Bag of Agents" problem:** Unstructured multi-agent coordination creates 17x error amplification. Agents echo and validate each other's mistakes.

**Recovery patterns:**
- State persistence + idempotent operations → rollback capability
- Graceful failure → stop, defer, escalate rather than produce garbage
- Intent replay → serialize goals/progress, replay from last checkpoint
- Circuit breakers → timeouts, confidence thresholds, human fallback
- Git-based recovery → revert bad changes via descriptive commits

---

## Handoff Patterns

**Core principle:** Treat inter-agent handoffs like a public API. Structured, versioned, validated.

**What makes handoffs reliable:**
- Structured output schemas, not prose
- Input filtering — receiving agent accepts only what it needs
- Context pairing — tool calls paired with their responses
- Version control on handoff formats

**MetaGPT's SOP approach:** Each role produces a specific output format that prompts the next role. PRD → Technical Design → Code → Tests. Standardized outputs establish explicit dependencies.

**Anthropic's subagent config:** Four elements per subagent — objective, output format, tool guidance, task boundaries. Early iterations failed with vague instructions like "research the semiconductor shortage."

---

## Key Anti-Patterns

1. **Bag of Agents** — unstructured multi-agent = 17x error amplification
2. **Free-form communication** — forces guesswork about intent
3. **Shared mutable state** — race conditions, state depends on write timing
4. **Vague delegation** — causes overlapping work, wasted tokens
5. **Skipping verification** — 25% of all failures
6. **Premature victory** — agents declare success before work is done
7. **Loading everything everywhere** — context rot degrades capability
8. **Monolithic work items** — too large = agent loses focus
9. **No error artifacts** — silent failures leave no recovery path
10. **Prompt-only fixes** — structural changes beat prompt tweaks (+5-14% vs major gains)

---

## Sources

### Anthropic
- Effective Context Engineering for AI Agents
- Effective Harnesses for Long-Running Agents
- Multi-Agent Research System (90.2% improvement, 90% time reduction)

### Failure Research
- MASFT taxonomy (150+ traces, 14 failure modes)
- 17x Error Trap of Bag of Agents (Towards Data Science)
- 12 Failure Patterns of Agentic AI (Concentrix)

### Frameworks
- LangGraph persistence/checkpointing
- MetaGPT SOP-based assembly line
- CrewAI role-based pipelines
- Claude Code subagent/worktree system

### Context & Memory
- Context Rot (Chroma Research, 18 models)
- Context Management for Deep Agents (LangChain)
- Filesystem-Based Agent State (Agentic Patterns)

---

*Research completed: 2026-02-26*
