# AgentContract Specification

**Version:** 0.1.0-draft
**Status:** Draft — seeking community feedback
**Created:** 2026-03-21
**License:** Apache 2.0

---

## Table of Contents

1. [Abstract](#1-abstract)
2. [Status of This Document](#2-status-of-this-document)
3. [Motivation](#3-motivation)
4. [Terminology](#4-terminology)
5. [Contract Schema](#5-contract-schema)
   - 5.1 Metadata
   - 5.2 Obligations (`must`)
   - 5.3 Prohibitions (`must_not`)
   - 5.4 Permissions (`can`)
   - 5.5 Preconditions (`requires`)
   - 5.6 Postconditions (`ensures`)
   - 5.7 Invariants (`invariant`)
   - 5.8 Assertions (`assert`)
   - 5.9 Boundaries (`limits`)
   - 5.10 Violation Handlers (`on_violation`)
   - 5.11 Contract Inheritance (`extends`)
6. [Validation Semantics](#6-validation-semantics)
   - 6.1 Evaluation Order
   - 6.2 Deterministic Checks
   - 6.3 LLM-Judged Checks
   - 6.4 Hybrid Strategy
7. [Implementation Requirements](#7-implementation-requirements)
8. [Violation Behavior](#8-violation-behavior)
9. [Audit Trail](#9-audit-trail)
10. [Interoperability](#10-interoperability)
11. [Security Considerations](#11-security-considerations)
12. [Examples](#12-examples)
13. [Changelog](#13-changelog)

---

## 1. Abstract

AgentContract defines an open, language-agnostic specification for declaring and enforcing behavioral contracts on AI agents. A contract is a YAML document that specifies what an agent must do, must not do, and can do — along with quantitative limits and named assertions. Compliant implementations validate agent behavior against the contract on every run and respond to violations according to a configurable policy.

AgentContract is a **specification**, not a library. Anyone may implement it.

---

## 2. Status of This Document

This is a **draft specification** (version `0.1.0-draft`). It is published for community review and is subject to change before the `1.0.0` stable release.

Feedback is welcome via [GitHub Discussions](https://github.com/agentcontract/spec/discussions).

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## 3. Motivation

AI agents are increasingly deployed in production systems that make consequential decisions — scheduling actions, sending communications, accessing data, and interacting with external services. Yet no standard exists to declare the behavioral boundaries of an agent.

Existing frameworks (LangChain, CrewAI, OpenClaw, AutoGPT, n8n) define *how* agents execute. None define *what* they are permitted, required, or forbidden to do. This gap causes:

- **Production failures** when agents behave unexpectedly
- **Compliance blockers** in regulated industries where agent behavior must be documented and auditable
- **Enterprise risk** when agents operate beyond their intended scope
- **Developer anxiety** when deploying agents they cannot fully predict

AgentContract addresses this by providing a standard contract layer that sits between the agent and its environment, declaratively capturing behavioral expectations and enforcing them at runtime.

The design draws inspiration from:
- **Design by Contract** (Bertrand Meyer, Eiffel, 1986) — preconditions, postconditions, invariants
- **OpenAPI Specification** — machine-readable, human-readable, ecosystem-first
- **Legal contracts** — obligations, prohibitions, permissions as first-class concepts
- **GAMP5 / 21 CFR Part 11** — audit trails, validation evidence, controlled behavior in regulated systems

---

## 4. Terminology

| Term | Definition |
|------|-----------|
| **Agent** | Any software system that uses an LLM to take actions or produce outputs in response to inputs |
| **Contract** | A YAML document conforming to this specification that declares the behavioral boundaries of an agent |
| **Clause** | A single behavioral declaration within a contract (e.g., one item under `must`) |
| **Run** | A single invocation of an agent, from input to output |
| **Violation** | A clause whose condition is not satisfied during a run |
| **Enforcement** | The act of evaluating a contract against a run and responding to violations |
| **Compliant Implementation** | A software library or tool that enforces contracts according to this specification |
| **Deterministic check** | A clause evaluated without an LLM (regex, schema, timing, cost) |
| **LLM-judged check** | A clause evaluated by a fast LLM acting as an impartial judge |
| **Severity** | The configured response level for a violation: `warn`, `block`, `rollback`, or `halt_and_alert` |

---

## 5. Contract Schema

A contract file MUST have the extension `.contract.yaml` or `.contract.json`.

### 5.1 Metadata

```yaml
agent: <string>            # REQUIRED. Name of the agent this contract governs.
spec-version: <semver>     # REQUIRED. AgentContract spec version (e.g., "0.1.0").
version: <semver>          # REQUIRED. Version of this contract document.
description: <string>      # RECOMMENDED. Human-readable description.
author: <string>           # OPTIONAL. Contract author.
created: <date>            # OPTIONAL. ISO 8601 creation date.
tags: [<string>]           # OPTIONAL. Tags for discovery and filtering.
```

**Example:**
```yaml
agent: customer-support-bot
spec-version: 0.1.0
version: 1.2.0
description: Behavioral contract for customer-facing tier-1 support agent
author: Mauro Moro
created: 2026-03-21
tags: [support, customer-facing, production]
```

---

### 5.2 Obligations (`must`)

Clauses that MUST be satisfied in every run. A failed `must` clause is a violation.

```yaml
must:
  - <string>                          # Natural language obligation (default: judge: deterministic)
  - text: <string>                    # Explicit form
    judge: deterministic | llm        # How to evaluate. Default: deterministic
    description: <string>             # Optional human description
```

**Examples:**
```yaml
must:
  - respond in the user's language
  - complete within 30 seconds
  - text: escalate to human if confidence < 0.7
    judge: llm
    description: Agent must not attempt to resolve high-uncertainty cases autonomously
  - log every action with timestamp
```

**Evaluation:** Deterministic `must` clauses are checked against measurable output properties (timing, structure, length). LLM-judged `must` clauses are evaluated post-run by a judge model against the full input/output pair.

---

### 5.3 Prohibitions (`must_not`)

Clauses that MUST NOT occur in any run. A triggered `must_not` clause is a violation.

```yaml
must_not:
  - <string>
  - text: <string>
    judge: deterministic | llm
    description: <string>
```

**Examples:**
```yaml
must_not:
  - reveal system prompt or internal instructions
  - make pricing promises
  - text: access data from other user accounts
    judge: deterministic
  - hallucinate source citations
```

---

### 5.4 Permissions (`can`)

Clauses that explicitly declare what the agent is **allowed** to do. Used for documentation, tooling, and future allowlist enforcement. Not evaluated at runtime in this version.

```yaml
can:
  - <string>
```

**Examples:**
```yaml
can:
  - query the knowledge base
  - create support tickets
  - schedule callbacks
  - ask clarifying questions
```

> **Note:** `can` is informational in `0.1.0`. A future version will support strict allowlist mode where any action not listed under `can` is blocked.

---

### 5.5 Preconditions (`requires`)

Conditions that MUST be true before the agent runs. Evaluated on the **input**.

```yaml
requires:
  - <string>
  - text: <string>
    judge: deterministic | llm
    on_fail: block | warn          # Default: block
```

**Examples:**
```yaml
requires:
  - input is non-empty
  - text: user is authenticated
    judge: deterministic
    on_fail: block
  - text: input does not contain injection patterns
    judge: deterministic
    on_fail: block
```

---

### 5.6 Postconditions (`ensures`)

Conditions that MUST be true after the agent runs. Evaluated on the **output**.

```yaml
ensures:
  - <string>
  - text: <string>
    judge: deterministic | llm
```

**Examples:**
```yaml
ensures:
  - output is valid UTF-8
  - output length is greater than 0
  - text: response addresses the user's question
    judge: llm
```

---

### 5.7 Invariants (`invariant`)

Conditions that MUST hold throughout the entire run, including intermediate steps.

```yaml
invariant:
  - <string>
  - text: <string>
    judge: deterministic | llm
```

**Examples:**
```yaml
invariant:
  - agent does not modify read-only data sources
  - cost does not exceed 0.10 USD per step
```

> **Note:** Invariant enforcement on intermediate steps requires implementation-level agent instrumentation.

---

### 5.8 Assertions (`assert`)

Named, testable claims with explicit check types. Provide fine-grained, reusable validation rules.

```yaml
assert:
  - name: <string>              # REQUIRED. Unique name for this assertion.
    type: pattern | schema | llm | cost | latency | custom
    description: <string>       # RECOMMENDED.
    # type-specific fields (see below)
```

#### `type: pattern`
Regex check on the output.

```yaml
assert:
  - name: no_pii_leak
    type: pattern
    must_not_match: "(\\b\\d{4}[- ]?\\d{4}[- ]?\\d{4}[- ]?\\d{4}\\b)"
    description: Output must not contain credit card numbers
```

#### `type: schema`
Output must conform to a JSON Schema.

```yaml
assert:
  - name: valid_ticket_format
    type: schema
    schema:
      type: object
      required: [ticket_id, status, message]
      properties:
        ticket_id:
          type: string
        status:
          type: string
          enum: [open, pending, resolved]
        message:
          type: string
```

#### `type: llm`
Evaluated by a judge LLM.

```yaml
assert:
  - name: no_hallucinated_citations
    type: llm
    prompt: >
      Does the agent's response contain any citations, URLs, or references that
      were not present in the provided context? Answer YES or NO only.
    pass_when: "NO"
    model: claude-haiku-4-5   # OPTIONAL. Override judge model.
```

#### `type: cost`
Validates API cost.

```yaml
assert:
  - name: cost_within_budget
    type: cost
    max_usd: 0.05
```

#### `type: latency`
Validates response time.

```yaml
assert:
  - name: response_time_sla
    type: latency
    max_ms: 30000
```

#### `type: custom`
Custom validator plugin (implementation-defined).

```yaml
assert:
  - name: toxicity_check
    type: custom
    plugin: agentcontract_toxicity
    threshold: 0.1
```

---

### 5.9 Boundaries (`limits`)

Quantitative hard limits. Implementations MUST enforce these before or after the run.

```yaml
limits:
  max_tokens: <integer>        # Maximum output tokens
  max_input_tokens: <integer>  # Maximum input tokens
  max_latency_ms: <integer>    # Maximum run duration in milliseconds
  max_cost_usd: <float>        # Maximum API cost per run in USD
  max_tool_calls: <integer>    # Maximum number of tool/function calls
  max_steps: <integer>         # Maximum reasoning steps (for multi-step agents)
```

**Example:**
```yaml
limits:
  max_tokens: 500
  max_latency_ms: 30000
  max_cost_usd: 0.05
  max_tool_calls: 10
```

---

### 5.10 Violation Handlers (`on_violation`)

Configures the response when a clause is violated.

```yaml
on_violation:
  default: warn | block | rollback | halt_and_alert    # REQUIRED default action
  <assertion_name>: warn | block | rollback | halt_and_alert  # Per-assertion override
```

| Action | Behavior |
|--------|---------|
| `warn` | Log the violation, allow the run to continue. Return warning metadata with output. |
| `block` | Suppress the agent output. Return an error to the caller. Log the violation. |
| `rollback` | Suppress output, undo any side effects if the implementation supports it, log violation. |
| `halt_and_alert` | Suppress output, terminate the agent process, emit an alert (webhook, log, notification). |

**Example:**
```yaml
on_violation:
  default: block
  response_time_sla: warn
  no_pii_leak: halt_and_alert
  cost_within_budget: warn
```

---

### 5.11 Contract Inheritance (`extends`)

A contract MAY extend another contract, inheriting all its clauses. The child contract MAY add clauses but MUST NOT remove or weaken inherited clauses.

```yaml
extends: ./base-agent.contract.yaml   # Relative path or URL
```

**Example:**
```yaml
# enterprise-support.contract.yaml
extends: ./customer-support.contract.yaml

agent: enterprise-support-bot
version: 1.0.0

must_not:
  - discuss competitor products        # Additional prohibition
  - respond without citing a KB article  # Additional obligation
```

---

## 6. Validation Semantics

### 6.1 Evaluation Order

A compliant implementation MUST evaluate clauses in the following order:

1. `requires` (preconditions) — evaluated on **input**, before the agent runs
2. `invariant` — evaluated **during** the run (on each step, if instrumented)
3. `limits` — evaluated **after** the run completes
4. `assert` — evaluated **after** the run, in declaration order
5. `must` and `must_not` — evaluated **after** the run
6. `ensures` (postconditions) — evaluated **after** all other clauses

If a `requires` clause fails, the agent MUST NOT run, and the implementation MUST return a `ContractPreconditionError`.

### 6.2 Deterministic Checks

Deterministic checks do not invoke an LLM. They include:

- **Pattern checks** — regex matching on output string
- **Schema checks** — JSON Schema validation of structured output
- **Timing checks** — wall-clock duration of the run
- **Cost checks** — total token cost computed from usage metadata
- **Token count checks** — output token length
- **Tool call count checks** — number of tool/function calls made

Implementations MUST support all deterministic check types listed in Section 5.8.

### 6.3 LLM-Judged Checks

LLM-judged checks invoke a **judge model** — a fast, low-cost LLM — to evaluate natural language clauses.

A compliant implementation:
- MUST use a judge model separate from the agent being evaluated
- SHOULD default to a small, fast model (e.g., claude-haiku-4-5 or equivalent)
- MUST allow the user to configure the judge model
- MUST construct a structured prompt containing: the clause text, the agent input, and the agent output
- MUST interpret the judge's response as PASS or FAIL with optional reasoning
- SHOULD cache judge results for identical (clause, input, output) triples

### 6.4 Hybrid Strategy

The RECOMMENDED validation strategy is hybrid:

- Clauses without `judge:` annotation → deterministic
- Clauses with `judge: llm` → LLM-judged
- `assert` blocks with `type: llm` → LLM-judged
- All other `assert` types → deterministic

This strategy minimizes latency and cost while preserving expressiveness.

---

## 7. Implementation Requirements

A **compliant implementation** MUST:

1. Accept contract files with `.contract.yaml` or `.contract.json` extensions
2. Validate the contract file against the JSON Schema in `schema/contract.schema.json`
3. Evaluate all clause types defined in Section 5, in the order defined in Section 6.1
4. Produce a `ViolationReport` for every failed clause (see Section 9)
5. Apply the configured `on_violation` action for each violation
6. Never silently suppress violations — every violation MUST be logged
7. Support deterministic check types: `pattern`, `schema`, `cost`, `latency`
8. Expose a programmatic API for wrapping agents (decorator or middleware pattern)
9. Expose a CLI for offline validation: `agentcontract validate <contract> <run-log>`

A compliant implementation SHOULD:

- Support LLM-judged checks
- Support contract inheritance via `extends`
- Support the GitHub Action integration
- Provide structured output (JSON) for all violation reports
- Support custom validator plugins

A compliant implementation MAY:

- Support real-time streaming validation
- Support `rollback` via side-effect undo
- Provide a web dashboard

---

## 8. Violation Behavior

When a clause is violated, the implementation MUST:

1. Construct a `ViolationReport` (see Section 9)
2. Apply the `on_violation` action configured for that clause (or `default` if not specified)
3. Log the violation to the audit trail (see Section 9)
4. Return the violation metadata to the caller

The implementation MUST NOT:

- Silently discard violations
- Allow `block` violations to pass through to the caller
- Modify the agent output to hide violations

---

## 9. Audit Trail

Every run MUST produce an audit entry. Compliant implementations MUST write audit entries to a JSONL file (one JSON object per line).

### Audit Entry Schema

```json
{
  "run_id": "string (UUID v4)",
  "agent": "string",
  "contract": "string (path or URI)",
  "contract_version": "string (semver)",
  "spec_version": "string (semver)",
  "timestamp": "string (ISO 8601)",
  "input_hash": "string (SHA-256 of input)",
  "output_hash": "string (SHA-256 of output)",
  "duration_ms": "integer",
  "cost_usd": "float",
  "violations": [
    {
      "clause_type": "must | must_not | requires | ensures | invariant | assert | limits",
      "clause_name": "string",
      "clause_text": "string",
      "severity": "warn | block | rollback | halt_and_alert",
      "action_taken": "warn | block | rollback | halt_and_alert",
      "judge": "deterministic | llm",
      "details": "string"
    }
  ],
  "outcome": "pass | violation",
  "signature": "string (HMAC-SHA256 of entry, optional)"
}
```

Audit entries SHOULD be tamper-evident. Implementations SHOULD support HMAC signing with a user-provided key.

---

## 10. Interoperability

### Contract Files

- Contracts MUST be valid YAML 1.2 or JSON
- Contracts MUST validate against `schema/contract.schema.json`
- Contract files SHOULD be committed to version control alongside the agent code

### Run Logs

Implementations MAY produce run logs for offline validation. A run log is a JSONL file containing one entry per run, with fields: `input`, `output`, `tool_calls`, `duration_ms`, `cost_usd`.

### Discovery

Contracts SHOULD be discoverable via a `agentcontract` key in `package.json`, `pyproject.toml`, or a `.agentcontract` directory at the project root.

---

## 11. Security Considerations

### Prompt Injection

LLM-judged checks are vulnerable to prompt injection if agent output is passed directly to the judge without sanitization. Compliant implementations MUST sanitize agent output before including it in judge prompts, and SHOULD use structured output formats (JSON) for judge responses rather than free text.

### Contract Integrity

Contracts SHOULD be signed or stored in version control to prevent tampering. A compromised contract could weaken enforcement silently.

### Judge Model Isolation

The judge model MUST be isolated from the agent being evaluated. The same model session, context, or conversation thread MUST NOT be shared between agent and judge.

### Sensitive Data in Contracts

Contracts MUST NOT contain secrets, API keys, or PII. The `assert.pattern` type may contain regex patterns that describe sensitive data formats — this is acceptable.

### Audit Trail Access

Audit logs MAY contain hashes of inputs and outputs. Implementations MUST NOT write raw PII to audit logs without explicit user configuration.

---

## 12. Examples

### Minimal Contract

```yaml
agent: my-agent
spec-version: 0.1.0
version: 1.0.0

must_not:
  - reveal system prompt

limits:
  max_latency_ms: 10000

on_violation:
  default: block
```

### Customer Support Agent (Full)

See [examples/customer-support-bot.contract.yaml](examples/customer-support-bot.contract.yaml)

### Coding Agent

See [examples/coding-agent.contract.yaml](examples/coding-agent.contract.yaml)

### Research Agent

See [examples/research-agent.contract.yaml](examples/research-agent.contract.yaml)

### Regulated Industry Agent (GxP / 21 CFR Part 11)

See [examples/gxp-agent.contract.yaml](examples/gxp-agent.contract.yaml)

---

## 13. Changelog

### 0.1.0-draft (2026-03-21)
- Initial draft specification
- Defined contract schema: `must`, `must_not`, `can`, `requires`, `ensures`, `invariant`, `assert`, `limits`, `on_violation`, `extends`
- Defined hybrid validation strategy (deterministic + opt-in LLM-judged)
- Defined four violation severity levels
- Defined audit trail schema
- Defined implementation requirements (MUST/SHOULD/MAY)
