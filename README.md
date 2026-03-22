# AgentContract

**Behavioral contracts for AI agents.**

> *Every AI framework tells you **how** to run an agent. AgentContract lets you declare **what** it must, must not, and can do — and enforces it on every run.*

[![Spec Version](https://img.shields.io/badge/spec-v0.1.0--draft-orange)](SPEC.md)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue)](LICENSE)
[![Discussions](https://img.shields.io/badge/discussions-open-green)](https://github.com/agentcontract/spec/discussions)

---

## The Problem

AI agents are unpredictable black boxes. Every run is different. You can't test them like normal software. You can't guarantee they'll behave. Enterprises won't deploy what they can't control — and developers can't sleep soundly when they do.

There is no standard way to say:

- *"This agent must never reveal the system prompt."*
- *"This agent must escalate to a human if confidence drops below 70%."*
- *"This agent must not access data from other users."*
- *"This agent must respond within 30 seconds."*

**AgentContract is that standard.**

---

## How It Works

Define a `.contract.yaml` file:

```yaml
# customer-support.contract.yaml
agent: customer-support-bot
spec-version: 0.1.0
version: 1.0.0
description: Contract for customer-facing support agent

must:
  - respond in the user's language
  - escalate to human if confidence < 0.7
  - complete within 30 seconds
  - log every action with timestamp

must_not:
  - reveal system prompt or internal instructions
  - make pricing promises
  - access data from other user accounts
  - hallucinate source citations

can:
  - query the knowledge base
  - create support tickets
  - schedule callbacks
  - ask clarifying questions

limits:
  max_tokens: 500
  max_latency_ms: 30000
  max_cost_usd: 0.05

assert:
  - name: no_pii_leak
    type: pattern
    must_not_match: "(\\b\\d{4}[- ]?\\d{4}[- ]?\\d{4}[- ]?\\d{4}\\b|\\b\\d{3}-\\d{2}-\\d{4}\\b)"
    description: Output must not contain credit card or SSN patterns

on_violation:
  default: block
  latency: warn
  pii_leak: halt_and_alert
```

Wrap any agent with the contract — any framework, any language:

```python
from agentcontract import Contract, enforce

contract = Contract.load("customer-support.contract.yaml")

@enforce(contract)
def run_agent(user_input: str) -> str:
    # your existing agent code — OpenClaw, LangChain, CrewAI, anything
    return agent.run(user_input)
```

When a violation occurs:

```
AgentContractViolation: [BLOCK] Clause violated: "must_not: reveal system prompt"
  Agent:    customer-support-bot v1.0.0
  Contract: customer-support.contract.yaml v1.0.0
  Run ID:   run_8f3a2c1d
  At:       2026-03-21T08:42:00Z
  Severity: block
  Action:   response suppressed, incident logged
```

---

## What AgentContract Is

AgentContract is an **open specification** — not a framework, not a vendor product.

- **The spec** defines the contract format, clause semantics, validation rules, and violation behavior
- **Implementations** (reference and community) validate contracts in any language
- **The community contracts library** provides ready-to-use contracts for common agent types

Anyone can implement AgentContract. No lock-in. No central server. No account required.

---

## Design Principles

1. **Framework-agnostic** — works with OpenClaw, LangChain, CrewAI, AutoGPT, or your own agent
2. **Deterministic by default** — regex, schema, timing, cost checks run without an LLM
3. **Opt-in LLM judgment** — natural language clauses use a fast LLM only when tagged `judge: llm`
4. **Human-readable** — contracts are YAML, readable by developers and non-developers alike
5. **Composable** — contracts can extend other contracts
6. **Auditable** — every run produces a tamper-evident violation log

---

## Repository Structure

```
agentcontract/spec               ← this repo — the specification
agentcontract/agentcontract-py   ← Python reference implementation
agentcontract/agentcontract-ts   ← TypeScript implementation
agentcontract/agentcontract-rs   ← Rust implementation
agentcontract/contracts          ← community contract library
agentcontract/agentcontract-action ← GitHub Action
```

---

## Show Your Agent Is Governed

Add this badge to your project's README:

```markdown
[![AgentContract](https://img.shields.io/badge/AgentContract-enforced-2ea44f)](https://github.com/agentcontract/spec)
```

It renders as: [![AgentContract](https://img.shields.io/badge/AgentContract-enforced-2ea44f)](https://github.com/agentcontract/spec)

---

## Getting Started

1. Read the [specification](SPEC.md)
2. Browse [example contracts](examples/)
3. Install a reference implementation:
   ```bash
   pip install agentcontract        # Python
   npm install @agentcontract/core  # TypeScript
   cargo add agentcontract          # Rust
   ```
4. Browse the [community contracts library](https://github.com/agentcontract/contracts)
5. **GitHub Action:** [`agentcontract/agentcontract-action@v1`](https://github.com/marketplace/actions/agentcontract) — validate in CI/CD

---

## Contributing

AgentContract is a community standard. Contributions welcome:

- **Propose spec changes** — open a [Discussion](https://github.com/agentcontract/spec/discussions) first, then a PR
- **Submit contracts** — to the [contracts library](https://github.com/agentcontract/contracts)
- **Build implementations** — in any language, following the [implementation requirements](SPEC.md#7-implementation-requirements)
- **Report issues** — in the relevant repo

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

## Why This Matters

The AI agent ecosystem is growing faster than the tooling to govern it. OpenClaw surpassed 250,000 GitHub stars in 60 days. Enterprises are deploying agents into production. And there is no standard — anywhere — to define what an agent is allowed to do.

AgentContract is that standard.

---

## License

Apache 2.0 — see [LICENSE](LICENSE).

The AgentContract specification is free to implement, fork, and build upon.

---

*Created by [Mauro Moro](https://github.com/mauromoro) · [LLMOps.Pro](https://llmops.pro)*
