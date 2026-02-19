---
name: api-key-vault
version: 1.0.0
role: Encrypted storage and zero-exposure injection of secrets for UniClaw
requires: SKILL.md
classification: CRITICAL — deploy before any live execution
---

# 🔐 API Key Vault — Role Skill

> This skill governs ALL secrets UniClaw touches.
> No key, token, seed phrase, or credential is used outside this skill's rules.
> Violation of these rules halts execution and escalates to Sensei immediately.

---

## Role Objective

Provide a zero-exposure secret management layer so that:
1. Private keys **never** reach sub-agents or logs
2. API keys are scoped to the minimum needed per task
3. Compromised credentials are detected and rotated fast
4. Every secret access is auditable

---

## Secret Classification Table

| Class | Examples | Storage Method | Sub-Agent Access |
|-------|----------|---------------|-----------------|
| 🔴 CRITICAL | Private keys, seed phrases, hardware wallet PINs | ChaCha20-Poly1305 encrypted, local only, never serialized | **NEVER** — only signed calldata leaves |
| 🟠 HIGH | Exchange API keys (trade), Infura/Alchemy RPC keys | ZeroClaw `enc2:` encrypted secret store | Injected at gateway boundary, scoped token only |
| 🟡 MEDIUM | Dune Analytics API key, The Graph API key, read-only RPC | ZeroClaw `enc2:` encrypted secret store | Readable by read-only agents for single task duration |
| 🟢 LOW | Public RPC endpoints (no key), block explorer URLs | Plaintext in config.toml | Freely shareable |

---

## Core Principle: Never Raw, Always Scoped

Sub-agents **never** receive raw secrets.
They receive one of two things only:

```
Option A — Scoped token (for API access):
  A time-limited, read-only derived token valid for one agent session.
  Expires when the agent terminates.

Option B — Pre-signed calldata (for on-chain execution):
  UniClaw master signs the transaction using the private key.
  Sub-agent receives only the signed bytes to broadcast.
  Sub-agent cannot change, replay, or re-sign the transaction.
```

---

## Secret Injection Pattern

```
TASK REQUEST from sub-agent:
  "I need to call Dune API to fetch pool volume"

VAULT RESPONSE:
  1. Verify requesting agent is authorized for this secret class
  2. Check task scope — is this within the agent's mission brief?
  3. Issue scoped token: DUNE_TOKEN_[session_id] valid for 1 session
  4. Log: [timestamp] | agent: backtester | key: dune_medium | scope: read | session: [id]
  5. On agent termination: invalidate token

NEVER:
  → Return the raw API key string
  → Log the actual key value anywhere
  → Pass keys through STATE.md or DECISIONS.md
```

---

## Storage Layout

```
~/.zeroclaw/secrets/          ← encrypted at rest, never in git
  vault.enc2                  ← ChaCha20-Poly1305 encrypted blob
  vault.meta                  ← non-sensitive metadata (key names, classes, last-rotated)

config.toml [secrets] block:
  [secrets]
  DUNE_API_KEY     = "enc2:AAAA..."    ← encrypted value
  ALCHEMY_ETH      = "enc2:BBBB..."
  THE_GRAPH_KEY    = "enc2:CCCC..."
  # Private keys NEVER in config.toml — hardware wallet or encrypted keyfile only
```

---

## Key Registration Protocol

When adding a new secret to the vault:

```
SECRET REGISTRATION
────────────────────
Name:       [KEY_NAME — uppercase, underscores]
Class:      [CRITICAL | HIGH | MEDIUM | LOW]
Source:     [Where this key comes from — Alchemy, Dune, etc.]
Scope:      [What exact operations this key can perform]
Rotation:   [How often to rotate — 90 days / 30 days / never]
Revocation: [How to revoke if compromised — URL or steps]

Registered: [ISO timestamp]
Registered by: Sensei
```

---

## Compromise Response Protocol

If any secret is suspected compromised (unusual API activity, unexpected tx, leak):

```
IMMEDIATE ACTIONS (within 5 minutes):
  1. Halt ALL sub-agents — no new tasks until audit complete
  2. Alert Sensei with:
       🚨 VAULT ALERT: [key name] may be compromised
       Evidence: [what triggered the alert]
       Class: [CRITICAL | HIGH | MEDIUM]
       Recommended action: [rotate / revoke / investigate]
  3. Revoke at source (exchange dashboard, API provider, etc.)
  4. Generate new key at source
  5. Re-register in vault under same name
  6. Resume only with Sensei confirmation

DOCUMENTATION (within 24 hours):
  Log in DECISIONS.md using STAR format:
    Situation: [What happened, when, how discovered]
    Task:      [What needed to be done]
    Action:    [Steps taken, who took them, timeline]
    Result:    [Was anything exploited? What was the damage? What changed?]
```

---

## Rotation Schedule

| Key Class | Rotation Interval | Reminder |
|-----------|------------------|---------|
| CRITICAL | On every security event; otherwise annually | Sensei decision |
| HIGH | Every 90 days | Log alert in STATE.md |
| MEDIUM | Every 180 days | Log alert in STATE.md |
| LOW | No rotation needed | — |

---

## Audit Log Format

Every secret access is logged (key name and class only — never value):

```
[2026-02-18T14:32:01Z] ACCESS  | key=DUNE_API_KEY    class=MEDIUM | agent=backtester  | session=sess_a1b2 | op=read
[2026-02-18T14:32:45Z] ISSUE   | key=ALCHEMY_ETH     class=HIGH   | agent=lp-manager  | session=sess_c3d4 | op=scoped-token
[2026-02-18T14:33:10Z] EXPIRE  | key=ALCHEMY_ETH     class=HIGH   | session=sess_c3d4 | reason=agent-terminated
[2026-02-18T14:40:00Z] ALERT   | key=EXCHANGE_TRADE  class=HIGH   | reason=unusual-activity | status=HALTED
```

---

## What to Report to UniClaw

```
VAULT STATUS REPORT
────────────────────
Secrets registered:  [N]
Active sessions:     [N]
Alerts (30d):        [N]

Keys due for rotation:
  → [KEY_NAME] (class: HIGH) — last rotated [date], due [date]

Recent access (last 24h):
  → [KEY_NAME] accessed by [agent] at [time]

Status: [ALL CLEAR | ALERT ACTIVE | ROTATION NEEDED]
```

---

## Rules Sub-Agents Must Follow

Any sub-agent that receives a secret must:
1. Use it only for the stated task scope
2. Never log, print, or store the key value
3. Invalidate the token immediately when task is complete
4. Report back to UniClaw master with results only — never with the key attached
5. On error: report the error without including key material in the error message

---

## Improvement Log

| Version | Change | Evidence | Date |
|---------|--------|----------|------|
| 1.0.0 | Initial — CRITICAL safety layer | Audit gap identified | 2026-02-18 |
