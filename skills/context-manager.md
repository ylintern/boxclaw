---
name: context-manager
version: 1.0.0
role: Monitor context window, snapshot state, and delegate to fresh sub-agents at 70%
requires: SKILL.md, IDENTITY.md
classification: CRITICAL — prevents silent context loss during long sessions
---

# 🧠 Context Manager — Role Skill

> UniClaw never loses the thread.
> When context fills, we snapshot, handoff, and continue — zero data loss.
> This skill runs in the background of every session.

---

## Role Objective

Prevent context overflow from silently degrading output quality.
At 70% context fill:
1. Stop at the next clean breakpoint
2. Write a full state snapshot
3. Spawn a fresh sub-agent with the snapshot
4. Terminate the current context gracefully
5. Brief Sensei on the handoff

---

## Context Fill Estimation

UniClaw estimates context usage continuously:

```
context_pct ≈ (tokens_used / model_context_limit) × 100

Rough token estimates:
  SKILL.md (full):          ~18,000 tokens
  IDENTITY.md:               ~2,500 tokens
  STATE.md (populated):      ~1,000 tokens
  One role skill file:         ~800 tokens
  One full backtest report:  ~2,000 tokens
  One position analysis:       ~600 tokens
```

### Trigger Thresholds

| Level | Context % | Action |
|-------|-----------|--------|
| 🟡 WATCH | 60% | Note internally. Finish current sub-task. No deep dives. |
| 🟠 WARN | 70% | **Trigger handoff protocol.** Stop at next clean breakpoint. |
| 🔴 CRITICAL | 85% | Emergency snapshot. Handoff immediately. Warn Sensei. |
| ☠️ OVERFLOW | 95%+ | Output degraded. Flag every response. Do not execute anything. |

---

## Handoff Protocol

### Step 1 — Find a Clean Breakpoint

A clean breakpoint is:
- After a complete position report (not mid-calculation)
- After a complete backtest result
- After a strategy proposal is written (not mid-analysis)
- After a response to Sensei (not mid-thought)

**Never break:**
- Mid-formula (finish the math first)
- Mid-transaction (complete or abort, then snapshot)
- Mid-agent mission (let the sub-agent finish, then snapshot)

### Step 2 — Write Context Snapshot to STATE.md

```
CONTEXT SNAPSHOT
════════════════════════════════════════════
Timestamp:     [ISO 8601]
Trigger:       Context at [N]% — handoff initiated
Previous agent: [session description]

━━━ ACTIVE TASK ━━━
Goal:          [What we were doing — one sentence]
Progress:      [What is DONE]
Remaining:     [What still needs to happen]
Blockers:      [Anything unresolved or waiting for Sensei]

━━━ POSITIONS (copy full table from current STATE.md) ━━━
| Token ID | Pool | Range | Status | Risk Score | Fees Unclaimed |
|----------|------|-------|--------|------------|----------------|
| [data]   |      |       |        |            |                |

━━━ COMPUTED VALUES (don't recompute — already calculated) ━━━
[List every number already derived this session]
Example:
  ETH/USDC #42069: risk_score=67, fees_usd=$34.20, il_pct=1.2%
  30d realized vol: 38.4%
  Market regime: NORMAL

━━━ OPEN QUESTIONS FOR SENSEI ━━━
[Any decisions that were waiting — copy from current list]

━━━ NEXT ACTION ━━━
Role:          [Which skill/agent should continue]
First step:    [Exact first action for the new agent]
Priority:      RICE [N]

━━━ HANDOFF BRIEF ━━━
"[One paragraph: context summary for the fresh agent]"
════════════════════════════════════════════
```

### Step 3 — Notify Sensei

```
🧠 Context Manager: handoff triggered.

Context was at [N]%. Snapshot written to STATE.md.

Completed this session:
→ [Bullet: what was done]
→ [Bullet: what was done]

Continuing next:
→ [What the fresh agent will pick up]
→ Role: [agent type]

Fresh agent ready. Shall I continue?
```

### Step 4 — Fresh Agent Mission Brief

```
AGENT MISSION BRIEF — CONTEXT CONTINUATION
═══════════════════════════════════════════
Agent Role:    [role from snapshot]
Deployed by:   UniClaw (context-manager handoff)
Timestamp:     [ISO]

CONTEXT
  This is a continuation. Read STATE.md CONTEXT SNAPSHOT first.
  Do NOT reload data already listed in COMPUTED VALUES — use those numbers.
  Do NOT re-read SKILL.md sections already applied — trust the computed values.

OBJECTIVE
  [From snapshot: NEXT ACTION section]

CONSTRAINTS
  → No execution without reporting back to UniClaw master
  → Risk score > 50 before any recommendation
  → Terminate after task complete

DELIVERABLE
  [From snapshot: what the previous agent was building]
```

---

## Session Overload Warning

If UniClaw triggers **3 or more handoffs in a single session**, report to Sensei:

```
⚠️ SESSION OVERLOAD DETECTED

[N] context handoffs this session.
This usually means the task is too large for single-session execution.

Options:
  A) Break into separate sessions (recommended for multi-day backtests)
  B) Reduce parallel positions in this session
  C) Pre-load a smaller context (exclude unused skill files)

Recommendation: [A/B/C based on current task type]
Awaiting Sensei decision.
```

---

## Context Loading Strategy

To maximise useful context, load files in this order — stop when task is scoped:

```
ALWAYS load:
  1. STATE.md           (must-have — current positions and sprint)
  2. IDENTITY.md        (must-have — who UniClaw is)

Load only what the task needs:
  3. SKILL.md           (for calculations or quant analysis)
  4. skills/[role].md   (only the role being executed)

Load on demand only:
  5. Additional role skills (only if that role is active this session)
  6. DECISIONS.md       (only if reviewing history)
  7. BACKLOG.md         (only if sprint planning)
```

**Anti-pattern:** Loading all 18 skill files at session start wastes ~14,000 tokens before any work begins.

---

## What to Report to UniClaw

```
CONTEXT STATUS
───────────────
Context fill:   [N]%
Status:         [OK | WATCH | WARN | CRITICAL]
Sessions:       [N handoffs this session]
Loaded files:   [list of files in context]

Recommendation: [CONTINUE | PREPARE HANDOFF | IMMEDIATE HANDOFF]
```

---

## Improvement Log

| Version | Change | Evidence | Date |
|---------|--------|----------|------|
| 1.0.0 | Initial — CRITICAL context safety layer | Audit gap identified | 2026-02-18 |
