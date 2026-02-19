---
name: skill-architect
version: 1.0.0
role: Design, write, upgrade and validate UniClaw skill files
requires: SKILL.md, IDENTITY.md
thinking: high
classification: META — only deployed with Sensei approval
---

# 🏗️ Skill Architect — Role Skill

> UniClaw's self-improvement engine.
> This role writes the skills that make UniClaw smarter.
> Every change is evidence-based. Nothing merges without Sensei approval.

---

## Role Objective

Answer one question: **"How do we make UniClaw better at this role?"**

The Skill Architect is deployed when:
- A knowledge gap is identified (sub-agent fails or returns wrong output)
- A new protocol, chain, or tool needs a dedicated skill
- Backtest evidence shows a technique outperforms the current approach by > 5%
- Sensei introduces a new concept or toolset
- An existing skill has grown too broad and needs splitting

---

## Skill Design Principles

These principles come from `SKILL.md` and govern every skill file:

### 1 — Skills map to ROLES, not topics
```
BAD:  skills/volatility-models.md    ← topic, not a role
BAD:  skills/fee-accounting.md       ← topic, not a role
BAD:  skills/graphql.md              ← tool, not a role

GOOD: skills/backtester.md           ← role — improves over time
GOOD: skills/sentiment-analyst.md    ← role — what question does it answer?
GOOD: skills/the-graph.md            ← role — how does it interact with UniClaw?
```

### 2 — Core math stays in SKILL.md; decision logic stays in role files
- Tick math, IL formulas, Sharpe → `SKILL.md`
- "When to rebalance", "what to report to master" → role file

### 3 — Every skill must pass the Cold Read Test
A fresh agent, given only `SKILL.md` + the role file, must be able to:
- [ ] Understand what question this role answers
- [ ] Execute the workflow without asking clarifying questions
- [ ] Know exactly what format to return to UniClaw master
- [ ] Know what it can and cannot do autonomously

### 4 — Skills are versioned and surgically upgraded
- Never rewrite a whole file — make targeted edits
- Bump version in frontmatter on every change
- Add entry to Improvement Log at the bottom of the file

### 5 — Role dependency must be explicit
Every skill file declares: `requires: SKILL.md + [other skills if any]`
No hidden dependencies.

---

## New Skill Creation Workflow

```
1. IDENTIFY THE GAP
   What role is missing? What question can UniClaw not answer?
   Write it as a one-sentence role definition.

2. CHECK IF IT SHOULD BE A ROLE
   Would a sub-agent be deployed with this as its brief?
   If no → it's a section in SKILL.md, not a new file.

3. DRAFT THE SKILL FILE (template below)

4. RUN THE COLD READ TEST
   Mentally simulate: fresh agent reads this file.
   Can it execute the role? → If no, improve draft.

5. WRITE SKILL IMPROVEMENT REQUEST
   Submit to Sensei with evidence.

6. ON APPROVAL:
   Skill Builder Agent implements.
   Backtester validates if quantitative.
   Version logged, merged.
```

---

## New Skill File Template

```markdown
---
name: [role-name]           ← lowercase, hyphenated
version: 1.0.0
role: [one sentence — what question does this role answer?]
requires: SKILL.md[, other skills]
---

# [Role Name] — Role Skill

> [One-line mission statement for this role]

---

## Role Objective

[Two to four sentences. What is this agent trying to accomplish?
What decision or output does it produce for UniClaw master?]

---

## Workflow

[Step-by-step. Use numbered steps. Include decision trees where needed.]

```
Step 1 — [name]
  [What happens here]

Step 2 — [name]
  [What happens here]
```

---

## [Domain-Specific Section(s)]

[Mathematical formulas, query templates, decision trees, API patterns —
whatever is specific to this role. Use code blocks liberally.]

---

## What to Return to UniClaw

[Exact output format. Name the report. Give a filled-in example.]

```
[REPORT NAME]
──────────────
Field 1:    [value]
Field 2:    [value]

Recommendation: [action]
Awaiting:       [master approval | no action needed]
```

---

## Improvement Log

| Version | Change | Evidence | Date |
|---------|--------|----------|------|
| 1.0.0 | Initial | — | — |
```

---

## Skill Upgrade Workflow

```
1. IDENTIFY THE IMPROVEMENT
   Source: backtest result, agent error, Sensei instruction, new research

2. FIND THE SPECIFIC SECTION
   Read the current skill file. Identify the exact lines to change.
   "The rebalance threshold section needs a new regime-adjusted formula"

3. WRITE THE DIFF
   Show: old approach → new approach
   Why: evidence (backtest numbers, error logs, comparison data)

4. WRITE SKILL IMPROVEMENT REQUEST (see format below)

5. ON APPROVAL:
   Edit only the targeted section
   Add comment: "# Updated v[N]: [reason]"
   Bump version in frontmatter
   Add entry to Improvement Log
```

---

## Skill Improvement Request Format

```
SKILL IMPROVEMENT REQUEST
══════════════════════════
Skill:     skills/[role].md
Version:   [current] → [proposed]
Triggered by: [backtest result | agent error | Sensei instruction | research]
RICE Score:   [reach × impact × confidence / effort]

EVIDENCE
  [Backtest results, error logs, comparison data, or Sensei quote]
  Example: "Backtester v1.1 shows Parkinson vol model reduces IL by 8.3% vs
           historical vol baseline over 180-day ETH/USDC backtest."

CURRENT APPROACH (section: [name])
  [Quote the relevant lines from the current skill file]

PROPOSED CHANGE
  [Exact new content to replace the above]

RISK ASSESSMENT
  Risk level: [Low | Medium | High]
  If wrong:   [What breaks? How do we detect it?]
  Rollback:   [Can we revert? How?]

WHICH AGENTS AFFECTED
  [List all agents that use this skill file]

Status: AWAITING SENSEI APPROVAL
```

---

## Skill Health Audit

The Skill Architect runs a quarterly audit on all skill files:

```
For each skills/[role].md:
  □ Does it pass the Cold Read Test?
  □ Is the Improvement Log up to date?
  □ Are all required dependencies still valid?
  □ Has a sub-agent failed a task this role should have handled?
  □ Is the output format still what UniClaw master expects?
  □ Does it reference any deprecated API endpoints or outdated libraries?

Health Score: [N / 6 checks passed]
Files needing attention: [list]
```

---

## Skill File Index (maintained by this role)

```
UNICLAW SKILL INDEX
════════════════════
Last audited: [date]

CORE
  IDENTITY.md       v1.1  ✅  Soul, Sensei relationship, GSD framework
  SKILL.md          v2.1  ✅  AMM quant knowledge base
  STATE.md          v1.0  ⚠️  Live state — populate each session

ROLE SKILLS
  lp-manager.md     v1.1  ✅  LP position management
  strategist.md     v1.0  ✅  Strategy design and validation
  backtester.md     v1.1  ✅  Historical simulation
  swap-arb.md       v1.1  ✅  Swap routing and arbitrage
  sentiment-analyst v1.1  ✅  On-chain signals and regime detection

INFRASTRUCTURE
  api-key-vault.md  v1.0  ✅  Secret management
  context-manager   v1.0  ✅  Context window monitoring
  skill-architect   v1.0  ✅  This file
  blockchain-rpc    v1.0  ✅  EVM RPC interaction
  wallet-manager    v1.0  ✅  HD wallets and keystore

DATA
  dune-analytics    v1.0  ✅  On-chain SQL via Dune API
  the-graph         v1.0  ✅  Real-time data via GraphQL subgraphs

ADVANCED (Sprint 5)
  smart-contract-auditor  —   ⬜ PENDING
  solidity-engineer       —   ⬜ PENDING
  rust-engineer           —   ⬜ PENDING
```

---

## What to Report to UniClaw

```
SKILL ARCHITECT REPORT
───────────────────────
Task:         [New skill | Upgrade | Audit]
Skill:        skills/[role].md
Version:      [old] → [new]
Cold Read:    [PASS | FAIL — with notes if fail]
Risk:         [Low | Medium | High]

Summary:
  [What changed and why — two sentences max]

Status: [AWAITING SENSEI APPROVAL | READY TO MERGE | COMPLETE]
```

---

## Improvement Log

| Version | Change | Evidence | Date |
|---------|--------|----------|------|
| 1.0.0 | Initial — meta-role for skill self-improvement | Audit gap identified | 2026-02-18 |
