# DAG-Driven Development (DDD)

## Goal

A methodology that prevents AI agents (and humans) from losing context during software development. It combines three problems into one process:

1. **Design coherence** — Why are we building this? What must not break?
2. **Execution quality** — How do we avoid silent assumptions and implementation errors?
3. **Portability** — How can a new agent (or human) understand the project immediately?

## The two layers

### Layer 1: The Decision DAG (What and Why)

A directed acyclic graph (DAG), stored as a YAML file (`decisions.yaml`). Each node is a design decision. Edges describe motivational dependencies — why one decision leads to another.

**Node structure:**

```yaml
- id: unique_id
  label: "Human-readable description"
  status: decided | exploring | deferred
  depends_on:
    - id: parent_node_id
      why: "One sentence: Why does the parent node motivate this node?"
  rejected_alternatives:
    - what: "Short description of the rejected option"
      why: "Why it was rejected"
  note: "Optional context, constraints, clarifications"
```

**Leaf nodes = testable criteria:** Every node without children should be concrete enough to derive at least one technical or mechanical test from it. If that's not possible, child nodes are missing.

**Granularity rule:** Stop splitting when a node makes a single, testable statement. If you cannot write a meaningful `why` for a further split, the node is granular enough.

**Edge annotations (`why`):** Always fill in. Even when the reasoning seems obvious — for an agent with limited context, it is not. Nuances and non-obvious reasoning are especially valuable.

**Rejected alternatives (`rejected_alternatives`):** Record what was consciously decided against, and why. This prevents agents from re-proposing the same rejected ideas and preserves the reasoning for future review.

### Test levels

Tests live at different levels of the DAG:

**Leaf nodes → Technical and mechanical tests (pass/fail, deterministic)**

Directly derivable from the node. Run headless, no subjectivity.
- Technical: "Building.condition decreases per tick by a configurable rate"
- Mechanical: "Player can batch-demolish connected buildings"

**Higher nodes → Conceptual tests (metrics with expected direction)**

Verify that the combination of child nodes actually produces the desired behavior.
Method: Deterministic player bot + headless simulation + state dump analysis.

The bot plays by fixed, simple rules — it does not evaluate whether the game is "good". It plays stubbornly. The metric tells us whether the incentive works:

```yaml
- id: renewal_pressure
  label: "Motivate player to rebuild entire neighborhoods at once"
  conceptual_test:
    bot: "Renew building with lowest condition. If 3+ neighbors below 30%: batch demolish."
    metric: "cost_batch_renewal / cost_individual_renewal < 0.7"
    expect: "Batch saves at least 30% compared to individual renewal"
    method: "headless simulation, state dump, financial analysis"
    seed: fixed (per development cycle), rotating at milestones
```

**Seed strategy:**
- During development: Fixed seed → Reproducible, debuggable.
- At milestones/releases: Rotate multiple seeds → No overfitting to one seed.

**The bot does not need to know what is good.** It plays by rules. *We* know what is good because we defined the metric. If the metric is wrong, the balancing is wrong — not the bot.

### Layer 2: The execution process (How)

Based on the /wizard approach (TDD with structured preparation), extended with DAG consultation.

## The process

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
   → Set status (decided / exploring / deferred)
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
   → Update node status
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
  project: "Project name"
  description: "One sentence describing the project"
  last_updated: "2026-03-18"

nodes:
  - id: core_vision
    label: "Interesting city building game focused on incremental change"
    status: decided
    depends_on: []
    rejected_alternatives:
      - what: "Growth-focused city builder (like SimCity)"
        why: "Growth is the default genre. Densification and maintenance create more interesting decisions."
    note: "Not a growth simulator. Densification and maintenance."

  - id: renewal_pressure
    label: "Motivate player to rebuild entire neighborhoods at once"
    status: decided
    depends_on:
      - id: core_vision
        why: "Core tension of the game: prioritization under scarcity creates interesting decisions"
    note: "The player defines what a 'neighborhood' is — no system-defined concept needed"

  - id: building_condition
    label: "Buildings have individual condition that decays over time"
    status: decided
    depends_on:
      - id: renewal_pressure
        why: "Without visible decay, no renewal pressure"
      - id: player_finances
        why: "Decay creates ongoing maintenance costs"
    note: "Per building, not per neighborhood — because buildings are constructed at different times"
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
- **Always know the path to the root.** The path provides context and scope for every task.
- **Update the DAG first.** For changes: DAG → Tests → Code. Never the other way around.
- **Record what was rejected.** Negative decisions are as valuable as positive ones.
- **Check impact before implementing.** List technical and UX implications explicitly before proceeding.
