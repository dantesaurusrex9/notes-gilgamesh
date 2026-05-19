---
title: "Intelligent Agents"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# Intelligent Agents

[toc]

> **TL;DR:** An intelligent agent is anything that perceives its environment through sensors and acts upon it through actuators. The PEAS framework (Performance, Environment, Actuators, Sensors) is the canonical tool for specifying agents. Agents range from dead-simple condition-action tables (simple reflex) to architectures that learn, maintain internal world models, and reason about future goals — the spectrum Percy Liang calls Reflex → States → Variables → Logic.

## Vocabulary

**Agent** — any entity that perceives its environment and produces actions; formally, a function from percept histories to actions.

**Percept** — a single sensory input at one time step; the agent's instantaneous view of the environment.

**Percept sequence** — the complete history of percepts received so far; the full input domain of the agent function.

**Rational agent** — one that selects actions expected to maximize its performance measure given its percept sequence and built-in knowledge.

**PEAS** — design framework specifying Performance measure, Environment, Actuators, Sensors.

**Performance measure** — an external criterion that evaluates how well the agent is doing; defined over environment states or outcome sequences, not the agent's internal state.

**Omniscience** — knowing the actual outcome of every action; distinct from rationality (which only requires maximizing *expected* performance given available information).

**Learning agent** — an agent that can improve its own behavior over time by modifying its performance element based on a critic's feedback.

**Agent program** — the concrete implementation of the agent function, running on the agent architecture (CPU, sensors, actuators).

---

**Simple reflex agent** — maps the current percept directly to an action using condition-action rules; no memory.

**Model-based reflex agent** — maintains internal state tracking the part of the world not visible from the current percept; uses a transition model to update that state.

**Goal-based agent** — augments the model-based agent with explicit goals; selects actions that lead to goal states via search or planning.

**Utility-based agent** — replaces the binary goal test with a utility function; enables rational behavior when goals conflict or states differ in quality.

---

**Fully observable** — the agent's sensors give complete access to the environment state at each step (vs. *partially observable*).

**Deterministic** — next state is fully determined by current state and action taken (vs. *stochastic*).

**Episodic** — experience is divided into independent episodes; actions in one episode do not affect later episodes (vs. *sequential*).

**Static** — environment does not change while the agent is deliberating (vs. *dynamic*).

**Discrete** — finite number of distinct states, percepts, and actions (vs. *continuous*).

**Single-agent** — only one agent operating in the environment (vs. *multi-agent*, where others may be cooperative or adversarial).

---

## Intuition

Think of an agent as a control loop: observe, think, act — then repeat. A thermostat sits at the simple end: no memory, no model, just "if temperature < setpoint then turn on heat." A self-driving car sits at the complex end: it tracks objects across frames, predicts trajectories, evaluates a utility function over safety and arrival time, and continuously learns from fleet data.

The key insight is the separation of concerns between *what* the agent knows about the world (its model) and *what* it is trying to achieve (its goal or utility). Rational action requires both — you cannot plan without knowing what actions do, and you cannot evaluate plans without knowing what counts as success.

Percy Liang's intelligence spectrum makes this precise:

```
Reflex → States → Variables → Logic
"Low-level"              "High-level"
```

Search problems, adversarial games, and Markov decision processes sit in the *States* tier. Constraint satisfaction and Bayesian networks sit in the *Variables* tier. Knowledge-based and logical agents sit at the top.

## How it works

An agent interacts with its environment through a perceive-act cycle that repeats at every time step. The agent program receives a percept and returns an action; the environment transitions to a new state and produces the next percept. The agent architecture (hardware + OS) executes the program.

### The PEAS framework

PEAS forces the designer to make four decisions explicit before writing any code. Skipping this step is the most common cause of misspecified agents that optimize the wrong objective. Two concrete examples:

| Domain | Performance | Environment | Actuators | Sensors |
| :--- | :--- | :--- | :--- | :--- |
| Medical diagnosis | Patient health, cost | Hospital, patient | Display queries, tests, treatments | Patient history, test results, vital signs |
| Self-driving car | Safety, speed, legality, comfort | Roads, traffic, pedestrians | Steering, throttle, brakes, horn | Cameras, LIDAR, GPS, odometry, speedometer |

> [!IMPORTANT]
> The performance measure is defined *externally* by the designer, not internally by the agent. An agent that maximizes its own happiness is not rational unless happiness correlates with the intended objective.

### Simple reflex agents

A simple reflex agent applies condition-action rules directly to the current percept, ignoring history. It is correct only when the environment is fully observable and Markovian — meaning the current percept contains all information needed to choose the best action. When the world is partially observable, a simple reflex agent gets stuck in loops because it cannot distinguish states that look identical from the current percept.

```python
from typing import Any

def simple_reflex_agent(rules: list[tuple[Any, Any]], percept: Any) -> Any:
    """
    rules: list of (condition, action) pairs, checked in order.
    Returns the action for the first matching condition.
    """
    for condition, action in rules:
        if condition(percept):
            return action
    raise ValueError(f"No rule matched percept: {percept}")

# Vacuum cleaner example: perceive (location, status)
rules = [
    (lambda p: p[1] == "Dirty", "Suck"),
    (lambda p: p[0] == "A",     "MoveRight"),
    (lambda p: p[0] == "B",     "MoveLeft"),
]

print(simple_reflex_agent(rules, ("A", "Dirty")))   # Suck
print(simple_reflex_agent(rules, ("A", "Clean")))   # MoveRight
```

### Model-based reflex agents

A model-based agent maintains internal state — a representation of the unobservable parts of the world. After each action and percept, it applies a *transition model* (how does the world evolve?) and a *sensor model* (what percepts does a given state produce?). Only then does it select an action. This breaks the Markov assumption requirement of the simple reflex agent: the internal state serves as a compressed history.

```python
from typing import Any

class ModelBasedAgent:
    def __init__(
        self,
        transition_model: Any,
        sensor_model: Any,
        rules: list[tuple[Any, Any]],
    ) -> None:
        self.state: Any = None
        self.transition_model = transition_model
        self.sensor_model = sensor_model
        self.rules = rules
        self.last_action: Any = None

    def act(self, percept: Any) -> Any:
        # Update internal state given last action and new percept
        self.state = self.update_state(self.state, self.last_action, percept)
        # Apply condition-action rules to updated state
        for condition, action in self.rules:
            if condition(self.state):
                self.last_action = action
                return action
        raise ValueError("No applicable rule")

    def update_state(self, state: Any, action: Any, percept: Any) -> Any:
        # In practice: apply transition model, then Bayesian update with sensor model
        return self.sensor_model(self.transition_model(state, action), percept)
```

### Goal-based agents

Goal-based agents extend the model with explicit goal descriptions — a set of desired world states or a predicate that returns True for goal states. The agent now performs *search* or *planning* to find a sequence of actions that reaches the goal. This is where uninformed and informed search algorithms become the engine of the agent program.

### Utility-based agents

A utility function U(s) assigns a real number to each state (or outcome sequence), encoding preferences among outcomes. Utility generalizes the binary goal test: it handles the case where multiple goal states differ in quality, and where goals conflict (fast vs. safe). A rational utility-based agent selects the action that maximizes *expected utility* over the distribution of possible outcomes.

```python
import random
from typing import Callable

def expected_utility(
    action: str,
    state: dict,
    outcomes: list[tuple[dict, float]],  # (outcome_state, probability)
    utility_fn: Callable[[dict], float],
) -> float:
    """Compute expected utility of taking an action from state."""
    return sum(prob * utility_fn(s) for s, prob in outcomes)

def utility_agent(
    actions: list[str],
    state: dict,
    outcome_model: Callable,
    utility_fn: Callable[[dict], float],
) -> str:
    """Select the action with highest expected utility."""
    return max(actions, key=lambda a: expected_utility(
        a, state, outcome_model(state, a), utility_fn
    ))
```

### Learning agents

A learning agent has four components: a *performance element* (what was previously the entire agent), a *critic* that evaluates behavior against a performance standard, a *learning element* that modifies the performance element based on critic feedback, and a *problem generator* that suggests exploratory actions that may yield better long-term performance. Reinforcement learning is the most general instantiation of this architecture.

## Math

The agent function is a mapping from percept sequences to actions:

```math
f : \mathcal{P}^* \to \mathcal{A}
```

where P* is the set of all finite percept sequences and A is the action set.

Rational action under uncertainty maximizes expected utility:

```math
a^* = \arg\max_{a \in \mathcal{A}} \sum_{s' \in \mathcal{S}} P(s' \mid s, a) \cdot U(s')
```

where P(s' | s, a) is the transition probability and U(s') is the utility of outcome state s'.

The performance measure over a sequence of T time steps is often defined as a sum of rewards:

```math
G = \sum_{t=0}^{T} r_t
```

or with discounting for infinite horizons:

```math
G = \sum_{t=0}^{\infty} \gamma^t r_t, \quad \gamma \in [0, 1)
```

## Real-world example

Consider the Wumpus World from the course slides: a 4×4 grid where an agent starts at [1,1] and must find gold (+1000) without falling in a pit (-1000) or being eaten by the Wumpus (-1000). The agent receives a 5-element percept vector [Stench, Breeze, Glitter, Bump, Scream].

This is a *knowledge-based agent* problem. A simple reflex agent fails immediately — adjacent squares to a breeze might have a pit, but which one? The agent needs to reason logically from percepts to safe squares. Below is a minimal Python skeleton showing the KB-AGENT loop from the lecture slides.

```python
from typing import Any

# KB-AGENT pseudocode (R&N Figure 7.1) in Python
class KBAgent:
    def __init__(self) -> None:
        self.kb: list[str] = []
        self.t: int = 0

    def tell(self, sentence: str) -> None:
        """Add a sentence to the knowledge base."""
        self.kb.append(sentence)

    def ask(self, query: str) -> bool:
        """Ask whether query is entailed by KB (stub: would run inference)."""
        # Real implementation: model checking or resolution
        return query in self.kb

    def make_percept_sentence(self, percept: list[Any], t: int) -> str:
        stench, breeze, glitter, bump, scream = percept
        return f"Percept({stench},{breeze},{glitter},{bump},{scream},t={t})"

    def make_action_query(self, t: int) -> str:
        return f"BestAction(t={t})"

    def make_action_sentence(self, action: str, t: int) -> str:
        return f"Action({action},t={t})"

    def act(self, percept: list[Any]) -> str:
        self.tell(self.make_percept_sentence(percept, self.t))
        action = self.ask(self.make_action_query(self.t))
        self.tell(self.make_action_sentence(str(action), self.t))
        self.t += 1
        return str(action)

agent = KBAgent()
# First percept: no stench, no breeze, no glitter
action = agent.act([None, None, None, None, None])
print(f"Step 0 action: {action}")
# Agent now has percept history in KB; logical inference over KB determines action
```

> [!TIP]
> In a real Wumpus agent, `ask()` runs propositional model checking over the KB. When KB has fewer than ~20 symbols, truth-table enumeration (2^n models) is tractable. Beyond that, use a SAT solver (e.g., `python-sat` or `z3`) rather than hand-rolling resolution.

## In practice

**Environment taxonomy drives architecture choice.** Before writing a single line of code, classify the environment on the six axes (observable, deterministic, episodic, static, discrete, single-agent). Partially observable + stochastic + sequential environments essentially require model-based agents with probabilistic state estimation (particle filters, Kalman filters). Trying to deploy a simple reflex agent in such an environment produces visible, catastrophic failures.

**The performance measure is a specification, not a reward signal.** In production ML systems the performance measure is implicit (a business metric) and the reward signal is a proxy (a logged label). The gap between these two is where Goodhart's Law lives. Rational agent theory formalizes the problem but does not solve the specification-reward gap — that requires careful reward modeling.

**Learning agents at scale require exploration–exploitation tradeoffs.** The problem generator component of a learning agent must balance exploiting current knowledge (greedily selecting the best known action) against exploring unknown regions of the state space. This is the multi-armed bandit problem in its simplest form and the core challenge of reinforcement learning.

> [!WARNING]
> Do not conflate *rational* with *omniscient*. A rational agent maximizes *expected* performance under uncertainty; it cannot be blamed for a bad outcome that results from genuinely unpredictable events. This distinction matters when evaluating real-world systems: an autonomous vehicle that correctly infers low probability of pedestrian and then observes a pedestrian was rational, not defective.

## Pitfalls

- **Wrong belief: a rational agent always achieves its goal.** Correction: rationality means maximizing expected performance given available information and computational resources — bad outcomes under uncertainty are consistent with rational behavior.

- **Wrong belief: the performance measure should be defined in terms of the agent's internal state (e.g., "agent believes it is healthy").** Correction: the performance measure is always defined over *environment* states or outcomes, not the agent's beliefs.

- **Wrong belief: a model-based agent always outperforms a simple reflex agent.** Correction: in fully observable, Markovian environments, the additional complexity of maintaining state is wasted. Match architecture to the environment.

- **Wrong belief: utility functions and reward functions are the same thing.** Correction: a utility function is a design-time specification over states; a reward function is a runtime signal the agent observes. They should agree, but in practice they often diverge — this divergence is the root cause of reward hacking.

- **Wrong belief: learning agents always improve.** Correction: a learning agent trained on a non-stationary environment can degrade. The learning element must account for distribution shift, or the critic's standard must be updated as the environment changes.

## Exercises

### Exercise 1

Classify the following environments on the six binary axes (observable, deterministic, episodic, static, discrete, single-agent):
- (a) A chess-playing program facing a human opponent.
- (b) A spam-filter that classifies individual emails.
- (c) A poker-playing agent in a five-player game.

#### Solution 1

**(a) Chess:**
- Observable: **fully** (both players see the entire board)
- Deterministic: **yes** (no randomness; moves have certain outcomes)
- Episodic: **no** (sequential; each move depends on the game history)
- Static: **yes** (board does not change while agent deliberates; turn-based)
- Discrete: **yes** (finite board, finite piece set, finite legal moves)
- Single-agent: **no** (two agents, adversarial)

**(b) Spam filter:**
- Observable: **fully** (filter sees entire email)
- Deterministic: **yes** (same email always produces same decision)
- Episodic: **yes** (each email is independent; classifying email N does not affect email N+1)
- Static: **yes** (email does not change during classification)
- Discrete: **yes** (finite token vocabulary)
- Single-agent: **yes**

**(c) Poker:**
- Observable: **partially** (cards in opponents' hands are hidden)
- Deterministic: **no** (card draws are stochastic)
- Episodic: **no** (sequential; stack size and opponent modeling persist across hands)
- Static: **yes** (table state does not change while an agent deliberates on their turn)
- Discrete: **yes** (finite deck, finite bet sizes in limit poker)
- Single-agent: **no** (multi-agent, mixed competitive-cooperative)

---

### Exercise 2

Design the PEAS specification for an automated essay-grading agent deployed in a university setting.

#### Solution 2

**Performance measure:** Pearson correlation between agent scores and expert human grader scores on held-out essays; per-rubric-criterion accuracy; false-positive rate for plagiarism flags.

**Environment:** Repository of submitted student essays (text, metadata: student ID, course, submission timestamp); rubric document; reference corpus for plagiarism detection; historical graded essays as training data.

**Actuators:** Grade output (numeric score per criterion), written feedback string, plagiarism flag (boolean), escalation signal to human grader.

**Sensors:** Essay text (UTF-8), rubric text, student metadata, embedding API (for semantic similarity), plagiarism database.

**Environment classification:** Fully observable (grader sees the full essay), deterministic, episodic (each essay is independent), static, discrete (grades are on a finite scale), single-agent.

---

### Exercise 3

A model-based vacuum cleaner agent has two rooms (A and B). It knows whether each room was clean or dirty at the last visit but it cannot directly observe the other room's current state. The world is *stochastic*: a cleaned room becomes dirty again with probability 0.1 between agent actions.

(a) What internal state does the agent need to maintain?
(b) Write the update equation for the internal state after observing (location=A, status=Clean) following action MoveLeft.

#### Solution 3

**(a)** The agent must maintain a probability distribution over the joint state of both rooms. Specifically, the internal state is a belief state — a distribution over {Clean, Dirty}² for rooms A and B:

```
b = P(room_A=clean, room_B=clean),
    P(room_A=clean, room_B=dirty),
    P(room_A=dirty, room_B=clean),
    P(room_A=dirty, room_B=dirty)
```

Four values summing to 1.

**(b)** After action MoveLeft from B (arriving at A), the transition model predicts that each clean room independently becomes dirty with probability 0.1. Then the observation (A=Clean) updates via Bayes' rule:

```math
P(\text{state} \mid \text{obs}) = \frac{P(\text{obs} \mid \text{state}) \cdot P'(\text{state})}{\sum_s P(\text{obs} \mid s) \cdot P'(s)}
```

where P'(state) is the prediction (post-transition, pre-observation) and P(obs | state) = 1 if obs is consistent with state, 0 otherwise (deterministic sensor). Concretely, all states where room_A = Dirty are eliminated, and the remaining probability mass is normalized.

---

## Sources

- Russell, S. & Norvig, P. *Artificial Intelligence: A Modern Approach*, 4th ed. Chapters 2 (Intelligent Agents). Pearson, 2020.
- Lecture slides: air(7).pdf — Intelligent Agents (University course materials, 2025).
- McCarthy, J. "Concepts of Logical AI." Stanford Formal Reasoning Group, 2000. http://www-formal.stanford.edu/jmc/concepts-ai/concepts-ai.html
- Liang, P. CS221 intelligence spectrum diagram. Stanford University.
- Conversation with user, 2026-05-19.

## Related

- [2 — Uninformed and Informed Search](./2-uninformed-and-informed-search.md)
- [3 — Adversarial Search and CSPs](./3-adversarial-search-and-csps.md)
- [4 — Logical Agents and Knowledge Representation](./4-logical-agents-and-knowledge-representation.md)
- [History of AI](../1-history-of-ai.md)
