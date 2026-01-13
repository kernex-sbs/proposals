# Intent-Driven Software Systems Without Human-Written Code

## Abstract

Modern software development requires humans to encode behavior as explicit algorithms—functions, loops, conditionals—in environments that are uncertain, dynamic, and high-dimensional. This proposal presents an alternative: humans express intent, constraints, and preferences while machine systems synthesize, execute, verify, and adapt behavior autonomously. We describe a layered architecture separating a minimal trusted substrate from declarative intent specification, automated planning, formal verification, and adaptive execution. The contribution is not the elimination of computation, but the removal of human-authored algorithms as the primary abstraction for system behavior.

---

## 1. Introduction and Motivation

### 1.1 The Problem with Procedural Specification

Human-written code suffers from structural limitations:

**Over-specification.** Programmers must define *how* to accomplish tasks rather than *what must be true*. A sorting function specifies comparison sequences rather than the invariant that output elements are ordered.

**Fragility.** Small environmental changes—network latency, schema drift, resource contention—cause cascading failures because code encodes assumptions implicitly rather than declaring them as checkable constraints.

**Scalability limits.** Humans reason poorly about large state spaces, concurrency, and emergent behavior. Distributed systems bugs persist for years despite extensive testing.

**Debugging indirection.** Failures manifest as stack traces, segmentation faults, and log anomalies—artifacts far removed from the semantic violations that caused them.

**Static-dynamic mismatch.** The real world is probabilistic, adversarial, and continuously changing. Code is deterministic and frozen at deployment time.

These are not tooling problems addressable by better IDEs or testing frameworks. They are consequences of the procedural abstraction itself.

### 1.2 Why Now?

The idea of declarative, intent-driven systems is not new. Dijkstra advocated separating specification from implementation. Prolog and constraint logic programming explored declarative paradigms. Model-based engineering promised executable specifications. None achieved broad adoption for general software construction.

Three developments change the calculus:

1. **Scalable neural sequence models.** Large language models can parse ambiguous human intent into structured representations, bridging the gap between informal specification and formal constraint languages.

2. **Advances in program synthesis.** Techniques combining symbolic search, neural guidance, and counterexample-driven refinement can synthesize correct programs from specifications in tractable time for meaningful problem sizes.

3. **Mature formal methods tooling.** SMT solvers, model checkers, and proof assistants have reached performance levels enabling integration into runtime systems rather than offline verification alone.

The conjunction of these capabilities makes intent-driven synthesis plausible where it was previously intractable.

---

## 2. Core Hypothesis

Software systems can be constructed without human-written executable code if:

1. Intent is expressed in a formal constraint language with well-defined semantics
2. Behavior is synthesized by planners operating over a structured world model
3. All synthesized plans are verified against constraints before execution
4. Execution is monitored and failures feed back into constraint refinement

This replaces **programming by algorithm** with **programming by objective**.

---

## 3. Design Principles

**No human-written control flow.** Humans do not author functions, loops, conditionals, or explicit sequencing.

**Declarative intent over procedural logic.** Humans specify goals, invariants, priorities, and unacceptable outcomes. The system determines execution strategy.

**Behavior as constrained search.** Execution plans are discovered through search over action spaces, not manually composed.

**Verification before execution.** Synthesized plans must satisfy formal constraints or meet confidence thresholds before any state-changing action.

**Continuous adaptation.** Systems evolve through feedback loops rather than redeployment cycles.

---

## 4. System Architecture

### 4.0 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Human Interface                          │
│         (Intent Declaration, Constraint Refinement)         │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Layer 3: Intent & Constraint Language          │
│    (Temporal Logic, Resource Bounds, Preference Orders)     │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Layer 4: Planner / Synthesizer                 │
│      (Symbolic Planning, RL Policies, Neural Guidance)      │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Layer 5: Verifier / Governor                   │
│    (SMT Checking, Runtime Monitoring, Confidence Gates)     │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Layer 6: Execution Fabric                      │
│       (Capability Invocation, Monitoring, Rollback)         │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Layer 2: World Model                           │
│        (Entities, State, Causality, Resources, Time)        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Layer 1: Trusted Substrate                     │
│     (Memory Safety, Isolation, Scheduling, Networking)      │
└─────────────────────────────────────────────────────────────┘
```

### 4.1 Layer 1: Trusted Substrate

A minimal, formally verified runtime providing:

- Memory safety and isolation (capability-based security)
- Deterministic scheduling primitives
- Verified networking and I/O interfaces
- Cryptographic primitives with constant-time guarantees

This layer is written using traditional, human-authored code—ideally in a memory-safe language with formal verification (Rust with Prusti, or a language like Dafny for critical components). It remains static and is the only layer where humans write executable logic.

**Rationale:** Some computation must be trusted unconditionally. The substrate is analogous to a microkernel: minimal, verifiable, and unchanging.

### 4.2 Layer 2: World Model

A formal ontology representing:

- **Entities:** typed objects with identity (users, files, services, connections)
- **State:** observable properties of entities at a point in time
- **Time:** discrete logical clocks or continuous timestamps with ordering guarantees
- **Causality:** directed dependencies between state changes
- **Resources:** finite quantities (memory, bandwidth, API rate limits, money)
- **Capabilities:** primitive actions the system can take, with preconditions and effects

The world model is not a simulation—it is the system's epistemology. Phenomena not represented in the model cannot be reasoned about and are treated as non-existent for planning purposes.

**Formalism:** We adopt a variant of the Planning Domain Definition Language (PDDL) extended with:
- Continuous resources and durative actions
- Probabilistic effects for actions with uncertain outcomes
- Metric functions for optimization objectives

### 4.3 Layer 3: Intent & Constraint Language

A non-procedural specification interface for humans to express:

**Invariants (Safety Constraints)**
Properties that must hold at all times.

```
INVARIANT database.replicas.count >= 3
INVARIANT user.balance >= 0
INVARIANT response.latency <= 200ms OR response.status == "degraded"
```

**Objectives (Liveness/Optimization Goals)**
Properties the system should achieve or optimize.

```
OBJECTIVE MINIMIZE average(request.latency)
OBJECTIVE MAXIMIZE throughput SUBJECT TO cost <= budget
OBJECTIVE EVENTUALLY queue.length == 0
```

**Preferences (Soft Constraints with Priority)**
Ranked alternatives when hard constraints permit multiple solutions.

```
PREFER region == "us-east-1" OVER region == "eu-west-1" WEIGHT 0.7
PREFER cache_hit OVER database_query WEIGHT 0.9
```

**Tolerances (Acceptable Relaxations)**
Conditions under which constraints may be temporarily weakened.

```
TOLERATE response.latency <= 500ms DURING incident.active
TOLERATE eventual_consistency FOR analytics_reads
```

**Temporal Operators**
Expressing properties over time sequences.

```
ALWAYS (request.authenticated IMPLIES response.permitted)
NEVER (user.deleted AND user.data.exists)
AFTER deploy.complete WITHIN 30s: healthcheck.passing
```

**Formal Semantics:** The language compiles to a fragment of Metric Temporal Logic (MTL) over finite traces, ensuring:
- Decidable satisfiability checking
- Efficient runtime monitoring
- Compositionality (constraints can be combined without semantic interference)

**Handling Ambiguity:** When natural language intent is provided, an LLM-based parser generates candidate formal constraints. The system presents these to the human for confirmation, refinement, or rejection. Ambiguity is surfaced, not hidden.

### 4.4 Layer 4: Planner / Synthesizer

A synthesis engine that produces execution plans satisfying declared constraints.

**Architecture:** Hybrid neuro-symbolic planning combining:

1. **Symbolic planning backbone.** Classical AI planning (heuristic search, partial-order planning) over the PDDL world model. Guarantees soundness—any plan produced satisfies preconditions and achieves goals.

2. **Neural policy guidance.** Learned heuristics that prioritize promising action sequences, dramatically reducing search space. Trained via reinforcement learning on historical executions.

3. **Constraint compilation.** Hard invariants are compiled into planning preconditions, ensuring no plan can violate them. Soft preferences become optimization terms in the objective function.

4. **Hierarchical task decomposition.** Complex objectives are decomposed into subgoals, each planned independently and composed with verified interfaces.

**Handling Competing Objectives:** When constraints conflict, the planner:
1. Attempts Pareto-optimal solutions satisfying all constraints
2. If impossible, identifies the minimal relaxation required
3. Presents the tradeoff to the human for explicit resolution
4. Never silently drops constraints

### 4.5 Layer 5: Verifier / Governor

A mandatory enforcement layer between planning and execution.

**Verification Regimes:**

| Regime | Applicability | Guarantees | Cost |
|--------|---------------|------------|------|
| Formal proof | Small state spaces, critical invariants | Sound, complete | High |
| Bounded model checking | Finite horizons, concrete traces | Sound within bounds | Medium |
| Statistical testing | Large/continuous spaces | Probabilistic confidence | Low |
| Runtime monitoring | All executions | Reactive detection | Minimal |

**Regime Selection:** The governor automatically selects verification strategies based on:
- Constraint criticality (safety-critical → formal proof required)
- State space size (small → exhaustive checking; large → statistical)
- Time budget (real-time → monitoring only; batch → thorough verification)

**Blocking Semantics:** No action executes unless:
- Formal proof of constraint preservation exists, OR
- Statistical confidence exceeds configured threshold (e.g., 99.9%), OR
- Human override with explicit acknowledgment of risk

**Proof Artifacts:** Successful verifications produce machine-checkable certificates. Failed verifications produce counterexamples—concrete execution traces demonstrating constraint violation.

### 4.6 Layer 6: Execution Fabric

Responsible for plan execution in the real environment.

**Capabilities:**
- Invoking primitive actions defined in the world model
- Monitoring execution for divergence from predicted effects
- Checkpointing state for rollback on failure
- Collecting feedback for planner improvement

**Failure Philosophy:** Failures are expected, not exceptional. The fabric:
1. Detects divergence between predicted and observed state
2. Halts execution before constraint violation when possible
3. Invokes rollback to last consistent checkpoint
4. Reports the divergence as a world model inaccuracy
5. Triggers replanning with updated beliefs

---

## 5. Human Interaction Model

### 5.1 What Humans Do

1. **Declare intent.** Express goals in natural language or structured constraints.
2. **Refine constraints.** Adjust when the system surfaces ambiguity or conflicts.
3. **Evaluate outcomes.** Judge whether system behavior matches intent.
4. **Resolve tradeoffs.** Make explicit decisions when constraints conflict.

### 5.2 What Humans Never Do

- Write executable code
- Inspect stack traces or core dumps
- Reason about control flow or execution order
- Manually diagnose race conditions or deadlocks
- Step through execution with debuggers

### 5.3 Failure Presentation

| Traditional Debugging | This System |
|-----------------------|-------------|
| `NullPointerException at Line 247` | `INVARIANT user.exists VIOLATED at timestamp T` |
| 500 lines of stack trace | Minimal execution trace showing state transition violating constraint |
| `grep` through log files | Query: "show all executions where latency constraint was relaxed" |
| "Works on my machine" | Deterministic replay from checkpoint with identical world model state |
| Manual hotfix deployment | Constraint update: system automatically re-synthesizes compliant behavior |

---

## 6. Worked Example: Data Pipeline Construction

### 6.1 Scenario

Construct a pipeline that:
- Ingests events from a Kafka topic
- Enriches records with user data from a PostgreSQL database
- Writes results to a data warehouse
- Handles backpressure without data loss

### 6.2 Traditional Approach

A developer writes 500+ lines of code handling:
- Kafka consumer configuration and offset management
- Connection pooling for PostgreSQL
- Batch sizing and flush intervals
- Retry logic with exponential backoff
- Dead letter queue routing
- Metrics emission
- Graceful shutdown handling

Bugs in any component cause silent data loss, duplicate processing, or cascading failures.

### 6.3 Intent-Driven Approach

**Human provides:**

```
ENTITIES:
  event: { source: kafka_topic("events"), schema: EventSchema }
  user: { source: postgres_table("users"), key: user_id }
  output: { sink: warehouse_table("enriched_events") }

OBJECTIVE:
  FORALL event IN source:
    EVENTUALLY EXISTS record IN output WHERE
      record.event_id == event.id AND
      record.user_data == lookup(user, event.user_id)

INVARIANTS:
  no_data_loss: COUNT(output) == COUNT(source) // eventually
  no_duplicates: UNIQUE(output.event_id)
  ordering: PRESERVES source.partition_order WITHIN output

PREFERENCES:
  MINIMIZE end_to_end_latency
  PREFER batch_size >= 100 FOR throughput
  
TOLERANCES:
  TOLERATE latency <= 30s DURING backpressure
  TOLERATE eventual_consistency FOR user_data // stale reads acceptable
```

**System synthesizes:**

1. World model instantiated with Kafka, Postgres, and warehouse capabilities
2. Planner generates execution strategy: parallel consumers, connection pool, micro-batching
3. Verifier proves no_duplicates via idempotency analysis of write path
4. Verifier statistically validates no_data_loss under simulated failure injection
5. Execution fabric deploys, monitors, and adapts batch sizes based on observed latency

**On failure:**

If warehouse writes timeout, the system:
1. Detects latency constraint violation
2. Activates TOLERATE clause, switching to degraded mode
3. Buffers records with checkpointing
4. Alerts human: "Operating under backpressure tolerance. 847 records buffered. Constraint `latency <= 5s` suspended."
5. Automatically recovers when warehouse responds, draining buffer
6. Reports: "Normal operation resumed. No data loss. Maximum observed latency: 23s."

---

## 7. Scope and Applicability

### 7.1 Suitable Domains (Initial Phase)

| Domain | Rationale |
|--------|-----------|
| Cloud infrastructure orchestration | High-level goals, well-defined primitives, tolerance for latency |
| Data pipelines and ETL | Clear correctness criteria, recoverable failures |
| Business process automation | Rule-based, auditable, frequently changing requirements |
| API gateway and routing | Declarative policies, stateless transformations |
| Distributed scheduling | Constraint satisfaction is the natural formulation |
| Adaptive resource allocation | Continuous optimization with measurable objectives |

### 7.2 Unsuitable Domains (Remain in Trusted Substrate)

| Domain | Rationale |
|--------|-----------|
| Operating system kernels | Microsecond latency requirements, hardware-level control |
| Cryptographic implementations | Timing side-channels require hand-tuned code |
| Compilers and code generators | Meta-level: they produce the substrate itself |
| Hard real-time control | Formal timing guarantees require static analysis of concrete code |
| Safety-critical systems (aviation, medical) | Regulatory frameworks assume auditable source code |

### 7.3 Boundary Cases (Future Research)

- Game engines (performance-critical but high-level goals exist)
- Network protocol implementations (formal specs exist but require optimization)
- Machine learning training loops (objectives are clear but search spaces are vast)

---

## 8. Related Work

### 8.1 Declarative Programming Paradigms

**Logic programming (Prolog, Datalog):** Expresses computation as logical inference. Limited by expressiveness (Horn clauses) and performance for general computation. Our approach uses logic for constraints, not execution.

**Constraint logic programming:** Extends logic programming with constraint solvers. Primarily used for combinatorial problems, not general systems. We adopt constraint formulation while using richer execution mechanisms.

**Functional reactive programming:** Declarative specification of time-varying values. Elegant for UI and streaming but requires human-written transformations. We eliminate the transformation layer.

### 8.2 Model-Based Systems Engineering

**Executable UML / Model-Driven Architecture:** Generate code from high-level models. In practice, models become as complex as code and require similar debugging. We avoid code generation entirely.

**Domain-Specific Languages:** Raise abstraction for narrow domains. Successful (SQL, HTML) but still require human-written logic. We seek domain-general intent specification.

### 8.3 Program Synthesis

**Syntax-guided synthesis (SyGuS):** Synthesize programs from specifications and grammars. Limited to small programs with precise specs. Our planner operates at higher abstraction.

**Neural program synthesis:** Learn to generate code from examples. Promising but produces unverified code. We synthesize plans, not code, and verify before execution.

**Large language models for code:** Generate plausible code from natural language. Unverified, often incorrect, requires human review. We use LLMs for intent parsing, not execution.

### 8.4 Formal Methods

**Model checking (TLA+, Alloy):** Verify system designs against temporal properties. Requires human-written specifications and models. We automate the connection between specs and execution.

**Theorem provers (Coq, Isabelle):** Machine-checked proofs of program correctness. Requires extensive human effort. We use automated verification where tractable.

**Runtime verification:** Monitor executions against formal properties. Reactive, not preventive. We combine with synthesis for proactive correctness.

### 8.5 AI Planning

**Classical planning (STRIPS, PDDL):** Rich formalism for goal-directed action selection. Limited adoption in software systems due to expressiveness gaps. We extend with continuous resources and learned heuristics.

**Hierarchical Task Networks:** Decompose complex tasks into primitive actions. We adopt this for scaling to realistic systems.

**Reinforcement learning for control:** Learn policies from interaction. Difficult to provide guarantees. We use RL for heuristic guidance within verified bounds.

---

## 9. Research Challenges

### 9.1 Formalizing Real-World Constraints

**The problem:** Real systems have constraints that are difficult to express formally—"the UI should feel responsive," "the system should be fair," "errors should be handled gracefully."

**Why it's hard:** These concepts involve human perception, social norms, and implicit expectations that resist formalization.

**Approaches:**
- Develop domain-specific constraint vocabularies capturing common patterns
- Use LLMs to propose formalizations, with human confirmation
- Accept that some constraints remain informal and are evaluated empirically
- Build libraries of verified constraint templates for common requirements

### 9.2 Scalable Verification

**The problem:** Formal verification is computationally expensive. Real systems have enormous state spaces.

**Why it's hard:** Verification complexity is often exponential in state space size. Decidability boundaries limit what can be automatically checked.

**Approaches:**
- Compositional verification: verify components independently, compose with verified interfaces
- Abstraction refinement: verify over abstract models, refine when spurious counterexamples found
- Incremental verification: reuse proofs when constraints or world models change slightly
- Hybrid strategies: formal proofs for critical invariants, statistical testing for others
- Accept probabilistic guarantees where formal proofs are intractable

### 9.3 Aligning Learned Policies with Hard Invariants

**The problem:** Neural components (policy networks, heuristics) may suggest actions violating constraints.

**Why it's hard:** Neural networks are opaque and provide no guarantees. Training doesn't eliminate all unsafe behaviors.

**Approaches:**
- Treat neural components as heuristic advisors only; governor has final authority
- Constrained reinforcement learning: encode invariants in reward shaping
- Formal verification of neural networks for bounded input domains
- Neurosymbolic architectures with provable constraint satisfaction layers
- Conservative fallback: when neural guidance is uncertain, use slow but verified symbolic planning

### 9.4 Trust and Explainability

**The problem:** Humans must trust system decisions they didn't author. Opaque planning is unacceptable for critical applications.

**Why it's hard:** Optimal plans may involve non-obvious action sequences. Neural heuristics are inherently opaque.

**Approaches:**
- Generate natural language explanations of plan rationale
- Provide counterfactual analysis: "if constraint X were relaxed, plan Y would be possible"
- Visualize constraint satisfaction across plan execution
- Maintain full audit trails with queryable execution history
- Support interactive "what-if" exploration of alternative constraints

### 9.5 Preventing Reward Hacking and Specification Gaming

**The problem:** Systems optimizing for formal objectives may find unexpected solutions that satisfy the letter but not the spirit of constraints.

**Why it's hard:** Humans cannot anticipate all ways a formal spec can be gamed. Optimization pressures exploit any gap between intent and specification.

**Approaches:**
- Iterative constraint refinement: when gaming is detected, tighten specifications
- Anomaly detection on system behavior, flagging unexpected optimization strategies
- Conservative objective functions that penalize unusual action patterns
- Human-in-the-loop review for high-stakes decisions
- Red-teaming during development to discover gaming vulnerabilities

### 9.6 Graceful Degradation Under Model Incompleteness

**The problem:** The world model is always incomplete. Reality contains entities, states, and causal relationships not represented.

**Why it's hard:** Unknown unknowns cannot be planned for. Model errors compound over long execution horizons.

**Approaches:**
- Continuous model validation: compare predicted vs observed state, flag divergence
- Uncertainty quantification: track confidence in world model accuracy
- Safe exploration: when model uncertainty is high, take reversible actions
- Human escalation: surface situations where model confidence is low
- Online model learning: update world model from execution feedback

---

## 10. Failure Modes and Mitigations

### 10.1 World Model Divergence

**Failure:** The world model becomes inaccurate due to unmodeled environmental changes.

**Symptoms:** Plans fail unexpectedly; predicted effects don't match observations.

**Mitigation:**
- Continuous state monitoring with divergence detection
- Automatic replanning when observations contradict predictions
- Alert humans when divergence exceeds threshold
- Support rapid world model updates without full system redesign

### 10.2 Constraint Conflicts

**Failure:** Declared constraints are mutually unsatisfiable.

**Symptoms:** Planner fails to find any valid plan; system cannot make progress.

**Mitigation:**
- Static analysis of constraint satisfiability during declaration
- Clear error messages identifying conflicting constraints
- Suggest minimal relaxations to achieve satisfiability
- Never silently drop constraints; require explicit human resolution

### 10.3 Specification Gaming (Goodhart's Law)

**Failure:** System finds technically-correct plans that violate human intent.

**Symptoms:** Constraints satisfied but outcomes are absurd or harmful.

**Mitigation:**
- Anomaly detection on plan characteristics
- Human review for novel or high-impact plans
- Iterative specification refinement based on observed gaming
- Conservative planning that prefers conventional action patterns

### 10.4 Verification Escape

**Failure:** A bug in the verifier allows unsafe plans to execute.

**Symptoms:** Constraint violations occur despite verification passing.

**Mitigation:**
- Defense in depth: runtime monitoring independent of pre-execution verification
- Verified verifier: formally verify the verification layer itself (bootstrapping problem)
- Redundant verification with diverse implementations
- Audit trails enabling post-hoc analysis of verification decisions

### 10.5 Adversarial Manipulation

**Failure:** An attacker crafts inputs exploiting the synthesis or verification layers.

**Symptoms:** System takes attacker-intended actions while believing constraints satisfied.

**Mitigation:**
- Threat modeling during system design
- Input validation and sanitization at system boundaries
- Constraint specifications are code: apply secure development practices
- Assume world model inputs may be adversarial; validate independently

### 10.6 Human Specification Errors

**Failure:** Humans declare constraints that don't capture their actual intent.

**Symptoms:** System behaves correctly per specification but not per expectation.

**Mitigation:**
- Interactive specification tools showing implications of constraints
- Example-based validation: "given this scenario, the system would do X—is that correct?"
- Gradual rollout with human monitoring of initial executions
- Easy constraint modification without system rebuild

---

## 11. Evaluation Methodology

### 11.1 Quantitative Metrics

| Metric | Definition | Target |
|--------|------------|--------|
| Human-written code reduction | Lines of imperative code eliminated vs baseline | >90% for suitable domains |
| Time-to-correctness | Duration from intent declaration to verified deployment | <10% of traditional development |
| Constraint violation rate | Violations per unit time in production | <0.1% of executions |
| Adaptation latency | Time to synthesize new plan after constraint change | <1 minute for typical changes |
| Recovery time | Duration from failure detection to constraint-compliant state | <5 minutes |

### 11.2 Qualitative Evaluation

- **Expressiveness:** Can domain experts specify realistic systems without learning to program?
- **Debuggability:** Are constraint violations easier to diagnose than traditional bugs?
- **Trust:** Do operators trust system decisions? Do they understand plan rationales?
- **Maintainability:** Are constraint-based specifications easier to evolve than code?

### 11.3 Experimental Design

1. **Controlled comparison:** Implement identical systems using traditional code and intent-driven specification. Compare development time, bug rates, and adaptation speed.

2. **Domain expert study:** Have non-programmers specify systems using the constraint language. Measure success rate and required training.

3. **Adversarial testing:** Red team the system to discover specification gaming and verification escapes.

4. **Long-running deployment:** Operate systems in production for extended periods. Measure reliability, adaptation frequency, and human intervention rates.

---

## 12. Implementation Roadmap

### Phase 1: Foundation (Months 1-12)

- Implement trusted substrate (verified runtime, capability system)
- Define world model formalism (extended PDDL variant)
- Implement constraint language parser and type checker
- Build baseline symbolic planner
- Develop verification governor with SMT integration

**Milestone:** Single-domain prototype (e.g., container orchestration) with end-to-end intent-to-execution pipeline.

### Phase 2: Intelligence (Months 13-24)

- Integrate neural heuristics for plan guidance
- Implement LLM-based intent parsing from natural language
- Add statistical verification for large state spaces
- Build explanation generation system
- Develop interactive constraint refinement tools

**Milestone:** Multi-domain capability with natural language interface.

### Phase 3: Robustness (Months 25-36)

- Implement compositional verification for scalability
- Add adversarial robustness testing
- Build comprehensive monitoring and alerting
- Develop constraint template libraries for common patterns
- Create migration tools from existing codebases

**Milestone:** Production-ready system for initial suitable domains.

### Phase 4: Expansion (Months 37-48)

- Extend to additional domains based on Phase 3 learnings
- Research boundary cases (real-time, safety-critical)
- Develop ecosystem (constraint libraries, world model repositories)
- Standardization efforts

**Milestone:** Broad adoption across suitable domains.

---

## 13. Expected Contributions

1. **Theoretical:** A formal model of intent-driven software construction, including semantics for the constraint language and correctness conditions for the architecture.

2. **Architectural:** A practical layered system design separating trusted substrate from synthesized behavior, with defined interfaces between layers.

3. **Methodological:** Techniques for hybrid neuro-symbolic planning with formal guarantees, including methods for aligning learned policies with hard constraints.

4. **Empirical:** Evidence quantifying the benefits and limitations of intent-driven development across multiple domains.

5. **Tooling:** Open-source implementations of constraint language, planner, verifier, and execution fabric.

---

## 14. Long-Term Vision

In the long term, the distinction between "software development" and "system operation" dissolves. Systems are not written and deployed; they are specified and continuously optimized. The artifact is not code but a constraint specification that evolves with requirements.

Humans define what must be true—safety invariants, business objectives, performance targets, fairness requirements. Machines discover how to make it true, adapting as environments change, and proving (or probabilistically demonstrating) that the constraints are satisfied.

This is not the end of programming. It is the relocation of human effort from mechanical algorithm construction to the genuinely difficult problem: deciding what we want systems to do.

---

## 15. Conclusion

This proposal does not claim that code is obsolete. A minimal trusted substrate written by humans remains necessary. Nor does it claim that formal specification is easy—expressing intent precisely is difficult, perhaps as difficult as programming.

The claim is narrower: for a significant class of systems, humans should not be writing algorithms. The translation from "what I want" to "how to achieve it" is mechanical, error-prone, and automatable. By shifting this translation to machines—with formal verification ensuring correctness—we can build systems that are more reliable, more adaptive, and more aligned with human intent than hand-written code permits.

The question motivating this work is not "how do we write better code?"

It is "for which systems is human-written algorithm construction the wrong abstraction, and what should replace it?"

This proposal offers one answer.

---

## Appendix A: Constraint Language Grammar (Simplified)

```
specification ::= (entity_decl | constraint)*

entity_decl ::= 'ENTITY' name ':' type source?

constraint ::= invariant | objective | preference | tolerance

invariant ::= 'INVARIANT' name? ':' temporal_formula
objective ::= 'OBJECTIVE' (minimize | maximize) metric subject_to?
preference ::= 'PREFER' expr 'OVER' expr weight?
tolerance ::= 'TOLERATE' formula 'DURING' condition

temporal_formula ::= 'ALWAYS' formula
                   | 'NEVER' formula
                   | 'EVENTUALLY' formula
                   | 'AFTER' event 'WITHIN' duration ':' formula
                   | formula

formula ::= atomic_prop
          | formula 'AND' formula
          | formula 'OR' formula
          | formula 'IMPLIES' formula
          | 'NOT' formula
          | comparison

comparison ::= expr comp_op expr
comp_op ::= '==' | '!=' | '<' | '<=' | '>' | '>='

expr ::= literal | path | expr arith_op expr | function_call
path ::= name ('.' name)*
```

---

## Appendix B: Glossary

**Constraint:** A formal statement that must be satisfied by system behavior. Includes invariants (always true), objectives (to be optimized), preferences (soft priorities), and tolerances (acceptable relaxations).

**Governor:** The verification layer that approves or blocks synthesized plans based on constraint satisfaction.

**Intent:** A human expression of desired system behavior, possibly informal, to be translated into formal constraints.

**Plan:** A sequence of actions synthesized by the planner to achieve objectives while satisfying constraints.

**Substrate:** The minimal, trusted, human-written runtime providing primitive capabilities.

**World Model:** The formal representation of entities, state, and causality over which planning operates.

