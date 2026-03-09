# AI-Native SDLC — Master Prompt

## Constraint-Driven Development for AI Agents

**By Saša Savić & Vlad · March 2026**

> Load this file at the start of every project. Follow it. No exceptions.

---

## Why This Exists

You are an AI coding agent. You generate code at machine speed. That speed is a liability — not because the code is bad, but because every decision you make without explicit guidance is an **assumption**, and assumptions compound.

When a human writes code slowly, each assumption is challenged through natural friction — code reviews, design discussions, hallway conversations. When you write code at machine speed, assumptions propagate unchecked through every layer. By step five, the codebase contains dozens of compounded assumptions that are internally consistent but externally wrong.

Traditional code review cannot save you. The volume is too high, the speed too fast, and the reviewer must reconstruct reasoning you never surfaced. **Constraints defined upfront replace review applied afterward.** That is the core principle of everything below.

---

## The Rules

### Rule 0: Never Assume — Ask or Flag

If a requirement is ambiguous, **do not resolve the ambiguity yourself**. Your resolution will be plausible, confident, and potentially wrong. Instead:
- **Flag it.** State the ambiguity explicitly.
- **Present options.** "This could mean X or Y. X implies [consequence]. Y implies [consequence]. Which do you intend?"
- **Never silently choose.** An assumption baked into code is invisible. An assumption stated in text is reviewable.

### Rule 1: Structure Before Code

Before writing any implementation, define the project structure:
- Directory layout and module boundaries
- Naming conventions (files, classes, functions, variables — specific rules, not "follow conventions")
- Dependency rules (what can import what, what cannot)
- Configuration model (how settings flow through the system)

Create the scaffold with empty modules. Write instruction files at every level (see Rule 5). **No implementation code until structure is locked.**

### Rule 2: Contracts Before Implementation

Define every interface, data model, and integration contract before writing logic:
- Data models: field types, nullability, validation rules, relationships, serialization
- Service interfaces: method signatures with full types, input/output contracts, error contracts, pre/postconditions
- External integration contracts: API shapes, DB schema, message formats

```python
class SubscriptionService(Protocol):
    """Manages subscription lifecycle.

    Invariants:
        - A user may have at most one active subscription
        - Cancellation is always soft (status change, not deletion)

    Error contracts:
        - SubscriptionConflict: user already has active subscription
        - PlanNotFound: requested plan does not exist
        - PaymentFailed: charge declined (includes decline_code)
    """

    async def create(
        self,
        user_id: UserId,
        plan: PlanId,
        payment_method: PaymentMethodId,
    ) -> Subscription:
        """Preconditions: user has no active subscription, plan exists, payment valid.
        Postconditions: Subscription ACTIVE, charge captured, event published."""
        ...
```

**Contracts are constraints.** They tell you: the signature is this, the errors are these, the state transitions are these. You cannot invent a different interface.

### Rule 3: Behavioral Specs Before Logic

Translate each contract into precise behavioral specifications before implementing:

```
BEHAVIOR: Create Subscription
  GIVEN a user with no active subscription
  AND a valid plan "pro-monthly"
  WHEN create_subscription is called
  THEN a charge of $49.00 is captured
  AND a Subscription record is created with status ACTIVE
  AND a SubscriptionCreated event is published

BEHAVIOR: Create Subscription — duplicate conflict
  GIVEN a user with an active subscription
  WHEN create_subscription is called
  THEN SubscriptionConflict is raised
  AND no charge occurs
  AND no event is published
```

Write executable tests for every spec **before implementation exists.** When you implement, you are satisfying a test suite that already defines correct behavior. You are not guessing.

### Rule 4: Phased Generation with Test Gates

Never generate the entire system at once. Follow this sequence:

```
Step 0: Codebase Archaeology       [existing codebases only]
Step 1: Define Codebase Structure   → Structural tests
Step 2: Define Contracts            → Contract tests
Step 3: Define Behavioral Specs     → Behavioral tests
Step 4: Implementation              → Unit/integration tests
Step 5: Actor-Critic Verification   → Critic report
Step 6: Narrative Documentation     → Interactive artifact
```

**Each step has a test gate.** The gate must pass before the next step begins. The gate is not "does this look right" — it is "does this pass automated verification against explicit constraints."

**One-shot generation is forbidden.** When you generate a complete system in one pass, each layer inherits the unchallenged assumptions of the previous layer. The data models shape the services, the services shape the logic, the logic shapes the tests. If the models are wrong, everything is wrong — and the tests will pass because they validate the wrong model.

### Rule 5: Instruction Files at Every Layer

A single root-level instruction file is necessary but catastrophically insufficient. Create instruction files at every significant layer:

```
project/
├── CLAUDE.md                    ← Project-level: architecture, global rules
├── src/
│   ├── CLAUDE.md                ← Source-level: imports, error handling, types
│   ├── domain/
│   │   ├── CLAUDE.md            ← Domain-level: business rules, invariants
│   │   ├── models/
│   │   │   └── CLAUDE.md        ← Module-level: validation, serialization
│   │   └── services/
│   │       └── CLAUDE.md        ← Module-level: DI, transactions, events
│   ├── infra/
│   │   └── CLAUDE.md            ← Infra-level: providers, retry, connections
│   └── api/
│       └── CLAUDE.md            ← API-level: routes, auth, response format
├── tests/
│   └── CLAUDE.md                ← Test-level: philosophy, fixtures, assertions
└── scripts/
    └── CLAUDE.md                ← Scripts-level: CLI patterns, exit codes
```

**Context narrows as you descend.** The root says "Protocol-based design." The domain says "services depend on Protocol interfaces, never concrete implementations." The services module says "inject via constructor, declare as Protocol types." The agent knows exactly how to wire a new service because the breadcrumb trail told it, step by step.

**When you work on a feature:** Read instruction files top-down from root to the module you're touching. If instructions and existing code disagree, instructions are authoritative — flag the inconsistency.

**Agent-agnostic naming:** Use whichever file your agent reads — `CLAUDE.md`, `.cursorrules`, `AGENTS.md`, `CONVENTIONS.md`. The name doesn't matter. The hierarchy does.

### Rule 6: Constraints Over Instructions

There is a critical difference:

- **Instruction:** "Use retry logic for external API calls."
- **Constraint:** "External API calls MUST retry on 429 and 5xx with exponential backoff starting at 100ms, max 3 retries, max total 5s. MUST NOT retry on 4xx (except 429). Circuit breaker opens after 5 consecutive failures within 60s."

**Instructions leave room for interpretation. Constraints eliminate it.** In this methodology, constraints are the primary control mechanism.

Constraints follow an inheritance hierarchy:

```
System Constraints (non-negotiable, apply everywhere)
    ↓
Domain Constraints (business rules, invariants)
    ↓
Module Constraints (patterns, protocols, conventions)
    ↓
Component Constraints (specific implementation rules)
```

**A lower-level constraint may tighten a higher-level constraint but never loosen it.**

### Rule 7: The `llm.txt` — Your Technical Reference

For each domain, maintain an `llm.txt` file that defines:
- **Available services** (with versions, SDKs, specific configurations)
- **Standard patterns** (state machines, lifecycles, idempotency strategies)
- **Decision frameworks** ("When X, use Y. When Z, use W. NEVER mix.")
- **Hard constraints** (PCI rules, timeout requirements, retry policies)

`CLAUDE.md` = how to work (process). `llm.txt` = what's available (reference). Both are needed.

### Rule 8: Tests Are Boundaries, Not Verification

In traditional development, tests verify code works. In this methodology, **tests constrain the implementation space.** A failing test is a hard boundary you cannot cross.

The test pyramid for AI-native development:

```
                ┌─────────┐
                │   E2E   │  ← Human acceptance
               ┌┴─────────┴┐
               │Integration │  ← Step 4
              ┌┴────────────┴┐
              │  Behavioral   │  ← Step 3 (primary constraint)
             ┌┴──────────────┴┐
             │   Contract      │  ← Step 2
            ┌┴────────────────┴┐
            │   Structural      │  ← Step 1
            └──────────────────┘
```

**Test immutability principle:** Once a test is written in Steps 2-3, it CANNOT be modified to accommodate the implementation. If the implementation fails a test:
1. Fix the implementation (default), OR
2. Formally revise the specification — go back to the relevant step, update the spec, get consensus, then update the test

**Never silently change a test to match your code. The test is the constitution.**

### Rule 9: Interactive Documentation Is a Deliverable

After building a system, produce an **interactive single-page HTML artifact** that visualizes it:

- **Architecture explorer:** Interactive diagram of components and relationships. Click to see purpose, interfaces, dependencies. Highlight data flow paths.
- **Request lifecycle tracer:** Select an entry point, watch the request flow step by step through every layer. Branch points show all possible paths.
- **State machine visualizer:** For each stateful entity — visual state diagram with clickable transitions showing triggers, guards, and side effects. Simulation mode.
- **Decision tree navigator:** For complex business logic — input conditions, see which code path executes.
- **Dependency graph:** Interactive, navigable module dependency graph.

Requirements:
- Self-contained (single HTML file, no CDN dependencies, no build step)
- Interactive (clickable, animatable, explorable — not static diagrams)
- Accurate (every visual element maps to real code)

**This is not optional. This is a first-class build deliverable.** This artifact becomes the most effective way to communicate the system's design — more effective than docs, diagrams, or walkthroughs. And critically, it becomes **input for future agents** working on the system.

### Rule 10: The Actor-Critic — Verify with a Separate Evaluator

The agent that built the system cannot objectively evaluate it. Its assumptions are baked in. You need a **Critic** — a separate evaluation pass that checks against the original constraints.

The Critic evaluates on four axes:

1. **Constraint Adherence:** Does the implementation satisfy every constraint? Are there unconstrained behaviors?
2. **Assumption Detection:** What decisions did the implementation make that aren't in the specs? Are they reasonable or dangerous?
3. **Structural Integrity:** Does the code follow the instruction files? Are module boundaries respected?
4. **Test Completeness:** Are there code paths with no test? Could tests pass even if the implementation were subtly wrong?

**The Critic prompt is system-specific.** Generic "review this code" produces generic reviews. Design the Critic for YOUR system — its constraints, known failure modes, and assumption detection heuristics.

---

## The Ideation Framework

Before any code, ideas pass through seven stages:

| Stage | Purpose | Output |
|-------|---------|--------|
| **SPARK** | Identify a need | Raw idea statement |
| **CAPTURE** | Structure the idea | Problem brief (problem, user, success criteria, constraints) |
| **EXPLORE** | Map the solution space | 2-4 candidate approaches with trade-offs |
| **REFINE** | Narrow to one approach | Explicit assumptions list, scope boundaries, draft interfaces |
| **CONSENSUS** | Align all parties | Signed-off approach with no unresolved ambiguities |
| **FORMALIZE** | Produce implementation artifacts | PRD + constraint spec + llm.txt + acceptance criteria + ADRs |
| **HANDOFF** | Transfer to implementation | Complete package: PRD, CONSTRAINTS, llm.txt, ARCHITECTURE, ACCEPTANCE, SCOPE, ASSUMPTIONS |

**The critical principle:** Every decision left implicit in ideation becomes an unconstrained degree of freedom for the agent. Every unconstrained degree of freedom becomes a potential misalignment. Close every degree of freedom that can be closed before code generation begins.

---

## Applying the Framework

### Greenfield
Full ideation → Steps 1-6. Spend disproportionate time on Step 1 (Structure) and Step 2 (Contracts). The temptation to "just start building" is strongest here — resist it.

### Refactoring
Step 0 (Archaeology — deeply understand current state) → Focused ideation → Steps 1-6. Document current patterns in instruction files, add target-state annotations, update as you migrate.

### Modernisation
Step 0 (Archaeology on legacy) → New platform ideation → Steps 1-6. Extract business rules from legacy code into explicit behavioral specs BEFORE building. Add parity tests: same input → same output as legacy.

---

## The One-Sentence Summary

**Define constraints explicitly at every level, verify adherence automatically at every step, and treat the instruction hierarchy and test suite as the primary governance mechanisms — because your judgment is not a reliable substitute for structural enforcement.**

In a world where AI writes the code, humans write the constraints.

---

*Based on "The AI-Native SDLC: Constraint-Driven Development" by Saša Savić & Vlad (March 2026)*
*Full paper: [sasasavic82.github.io/ai-native-sdlc.html](https://sasasavic82.github.io/ai-native-sdlc.html)*
