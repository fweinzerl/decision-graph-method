# Decision Graph Method

## Goal

A methodology that prevents AI agents (and humans) from losing context during software development. It combines three problems into one process:

1. **Design coherence** — Why are we building this? What must not break?
2. **Execution quality** — How do we avoid silent assumptions and implementation errors?
3. **Portability** — How can a new agent (or human) understand the project immediately?

## The Decision DAG

A directed acyclic graph (DAG), stored as a YAML file (`decisions.yaml`). Each node is a design decision. Edges describe motivational dependencies — why one decision leads to another.

**The DAG has three kinds of nodes:**

- **Root nodes** — The premises. State what is being built, without justification. They are not decisions — they are the starting points from which all decisions follow. A DAG can have multiple independent roots. Root nodes have no `depends_on`, no `rejected_alternatives`, and no `why`.
- **Inner nodes** — Design decisions. They provide context, motivation, and structure. They have no status and no direct implementation.
- **Leaf nodes** — Implementable units. They are concrete enough to derive at least one test from. They carry a `status` field that tracks implementation progress.

Every node in the DAG represents a decision that has been made. Ideas, open questions, and undecided options do not belong in the graph.

**Inner node structure:**

```yaml
- id: unique_id
  label: "Human-readable description"
  depends_on:
    - id: parent_node_id
      why: "One sentence: Why does the parent node motivate this node?"
  rejected_alternatives:
    - what: "Short description of the rejected option"
      why: "Why it was rejected"
  note: "Optional context, constraints, clarifications"
```

**Leaf node structure:**

```yaml
- id: unique_id
  label: "Human-readable description"
  status: open | in_progress | done
  depends_on:
    - id: parent_node_id
      why: "One sentence: Why does the parent node motivate this node?"
  rejected_alternatives:
    - what: "Short description of the rejected option"
      why: "Why it was rejected"
  note: "Optional context, constraints, clarifications"
```

**Status** exists only on leaf nodes and tracks implementation:

- `open` — Not yet started
- `in_progress` — Currently being implemented
- `done` — Implemented and tests passing

The implementation state of an inner node is implicit: it follows from the status of its leaf descendants.

**Granularity rule:** A node is granular enough when it makes a single, directly implementable and testable statement. If it is not implementable as a single unit, it needs children — and becomes an inner node.

**Edge annotations (`why`):** Always fill in. Even when the reasoning seems obvious — for an agent with limited context, it is not. Nuances and non-obvious reasoning are especially valuable.

**Rejected alternatives (`rejected_alternatives`):** Record what was consciously decided against, and why. This prevents agents from re-proposing the same rejected ideas and preserves the reasoning for future review.

### Test levels

Tests live at different levels of the DAG:

**Leaf nodes → Direct tests (pass/fail, deterministic)**

Directly derivable from the node. Automated, no subjectivity.
- "Failed deliveries are retried up to 3 times with exponential backoff"
- "Retry intervals are read from the JSON config file"

**Inner nodes → Metric tests (indicators, not proofs)**

A single leaf test proves that one piece works. A metric test checks whether the *combination* of children actually produces the intended effect. The structure is always the same:

1. Define a **scenario** — deterministic, reproducible, automated
2. Define a **metric** with an expected direction
3. The scenario does not know what "good" is — the metric defines it

Examples:

| Goal (inner node) | Scenario | Metric |
|---|---|---|
| Reliable delivery | Send 1000 notifications under simulated provider failures | Delivery rate > 99.5% |
| Configurable by non-programmers | Hand a new user a config task + docs, no code access | Task completed within schema validation, no invalid states |
| Fast API response | Load test with 500 concurrent requests | P95 latency < 200ms |
| Data deduplication | Run pipeline on dataset with 5% known duplicates | Duplicate rate in output < 0.1% |

**These metrics are not proofs.** They are indicators. A passing metric does not guarantee correctness. A failing metric is a signal to investigate — something in the combination of children is not producing the intended behavior. The metric points to the area; the diagnosis is still human work.

**Reproducibility:** During development, use fixed test conditions (seeds, datasets, configurations) for debuggability. At milestones, vary them to ensure the design does not overfit to one specific scenario.

## The Process

### A. Making a new design decision

```
1. Consult the DAG
   → Where in the graph does the new idea belong?
   → Which existing nodes are affected?

2. Impact check
   → List the technical implications of the decision.
   → List the UX implications of the decision.
   → Present both lists explicitly and get confirmation before proceeding.

3. Adversarial review (design level)
   → Is there a better option? Briefly name 2-3 alternatives.
   → Does the decision contradict existing nodes?
   → Does it create unwanted dependencies?

4. Update the DAG
   → Add new nodes and edges
   → Write edge annotations
   → Set status on leaf nodes (open / in_progress / done)
   → Record rejected alternatives from step 3
```

### B. Implementing a feature

```
1. Consult the DAG
   → Which leaf node is being implemented?
   → Which path leads to the root? (= context and motivation)
   → Which sibling/neighbor nodes must not break?

2. Explore the codebase
   → Read existing code affected by the feature
   → List assumptions explicitly and get confirmation

3. Write tests (TDD)
   → Derive tests from the leaf node
   → The path to the root provides scope: What do I want to achieve, what may I touch?
   → Mutation testing mindset: Tests verify specific outcomes, not just "runs without error"

4. Implement minimally
   → Only the minimum to pass the tests
   → No scope creep, no premature abstraction

5. Regression tests
   → Run existing tests
   → Check neighbor nodes in the DAG: Are their tests still green?

6. Adversarial review (code level)
   → What happens in edge cases?
   → Does the implementation violate dependencies in the DAG?

7. Update DAG + documentation
   → Update leaf node status (open → in_progress → done)
   → Record new insights in edge annotations
```

### C. Making a design change

```
1. Identify affected nodes in the DAG
   → List all children and transitive dependents of the changed node
   → This is the impact radius

2. For each affected node, check:
   → Is it still valid?
   → Do tests need to be adjusted?
   → Do child nodes need to be removed or changed?

3. Update the DAG, then adjust tests, then adjust code
   → Always in this order
```

## DAG file format

The file `decisions.yaml` lives in the project root. Example:

```yaml
# decisions.yaml
meta:
  project: "NotifyHub"
  description: "Multi-channel notification service with templating and delivery tracking"
  last_updated: "2026-03-29"

nodes:
  # Root nodes (premises, no justification needed)
  - id: core
    label: "Modern notification service"
    depends_on: []

  - id: configurable
    label: "Configurable by non-programmers"
    depends_on: []

  # Inner nodes (design decisions, no status)
  - id: delivery_reliability
    label: "Every notification is delivered at least once"
    depends_on:
      - id: core
        why: "A notification service that loses messages has no reason to exist"
    rejected_alternatives:
      - what: "Exactly-once delivery"
        why: "Requires distributed transactions across external providers; disproportionate complexity"

  - id: json_config
    label: "Configuration via JSON files"
    depends_on:
      - id: configurable
        why: "JSON is structured enough for validation but readable without programming knowledge"
    rejected_alternatives:
      - what: "GUI for configuration"
        why: "Too much development effort for the current team size"

  # Leaf nodes (implementable, with status)
  - id: retry_with_backoff
    label: "Failed deliveries are retried with exponential backoff up to 3 times"
    status: done
    depends_on:
      - id: delivery_reliability
        why: "Transient provider failures are common; retries are the simplest path to at-least-once"
    rejected_alternatives:
      - what: "Infinite retries"
        why: "Permanent failures (invalid address) would retry forever and waste resources"

  - id: retry_config
    label: "Retry count and backoff intervals are configurable per channel"
    status: open
    depends_on:
      - id: retry_with_backoff
        why: "Different channels have different failure characteristics; one-size-fits-all limits aren't practical"
      - id: json_config
        why: "Retry tuning is a common operational need — must be changeable without code changes"
```

## Tools

### Phase 1: YAML + manual queries
- `decisions.yaml` in the project root
- Agent reads the file directly
- Human reads it in an editor

### Phase 2: CLI tool (when the graph grows)
- `dag show <node_id>` — Show node with all parents and children
- `dag path <from> <to>` — Path between two nodes
- `dag impact <node_id>` — All transitive dependents
- `dag leaves` — All leaf nodes (= implementable features)
- `dag untested` — Leaf nodes without assigned tests

### Phase 3: Graphical visualization (optional)
- Script generates Graphviz DOT from the YAML
- Rendering as SVG/PNG for overview
- Also displayable in terminal (e.g. via `graph-easy`)

## Portability

To apply DDD to an existing project:

1. Give this document to the agent
2. Agent reads existing documentation, README, CLAUDE.md
3. Agent creates initial DAG: root = project goal, first level = main features
4. Human reviews and corrects the DAG
5. From now on, the process applies to all changes

For a new project: The DAG grows organically with the design decisions during the interview process.

## Principles

- **DAG before code.** No implementation without a node in the DAG.
- **Explain edges.** Every dependency has a `why` annotation.
- **Leaves are testable.** If a leaf node implies no test, it lacks granularity.
- **Always know the path to a root.** The path provides context and scope for every task.
- **Update the DAG first.** For changes: DAG → Tests → Code. Never the other way around.
- **Record what was rejected.** Negative decisions are as valuable as positive ones.
- **Check impact before implementing.** List technical and UX implications explicitly before proceeding.
