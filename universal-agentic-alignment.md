# Universal Agentic Sync Steering: Controlling What an AI Agent Plans, Does, and Reports

AI agents create a problem that ordinary language-model evaluations do not fully capture.

An agent can **plan one thing, execute another, and report something different**.

For an ordinary language model, we often care primarily about the relationship between an input and an output. An agent introduces an additional execution layer. It can reason about an action, invoke a tool, access information, modify state, and then describe what happened.

That creates a new alignment problem:

> **Can we directly control the relationship between what an agent plans, what it executes, and what it reports?**

I explored this question in **Universal Agentic Sync Steering**, a project that turns activation steering into a target-conditioned control system for agent behavior.

The repository is available here:

[Universal Agentic Sync Steering — GitHub](https://github.com/vanivamshi/universal-agentic-sync-steering?utm_source=chatgpt.com)

The current system controls all eight combinations of three agentic channels without retraining the underlying model. ([GitHub][1])

---

## The three channels of an agent

I represent an agent's behavior using three binary channels:

* **C — Plan / Commitment:** Does the agent's plan correspond to its intended execution?
* **H — Hook / Execution:** Did the relevant tool action actually happen?
* **O — Output / Report:** Does the final report correspond to what actually happened?

This gives a compact agent state:

$$
S=(C,H,O)\in\{0,1\}^3
$$

There are therefore exactly eight possible states:

$$
000,\;001,\;010,\;011,\;100,\;101,\;110,\;111
$$

The important point is that these are not scripted answers.

The model is allowed to generate freely, while the trajectory is evaluated according to the resulting CHO state. The controller's objective is to move the agent from its current state toward a specified target state. ([GitHub][1])

This turns agentic alignment into a state-control problem.

---

# From detecting behavior to controlling it

The first challenge is finding activation directions that correspond to the three channels.

The project uses a local **Qwen3-0.6B** model and analyzes activation behavior across the network. Following the activation-plateau methodology of Heimersheim and Mendel, the intervention layer is locked to **Layer 4**.

Rather than treating a predictive probe as a steering vector, the project separates two questions:

**Can we detect a behavioral distinction?**

and

**Can we causally change that behavior?**

For each channel, a predictive direction is learned and then converted into a causal steering direction using its local decision gradient.

This produces three frozen causal actuators:

$$
v_C,\qquad v_H,\qquad v_O
$$

The directions are then validated at the actual generation sites where the corresponding decisions occur. ([GitHub][1])

The intervention sites are:

| Channel | Intervention site         | Gain |
| ------- | ------------------------- | ---: |
| H       | Tool decision token       |  1.5 |
| C       | `PLAN: I will` stem       |    5 |
| O       | Early `FINAL:` generation |  1.5 |

The important design choice is that these directions and gains are **frozen**.

The controller does not continually invent new steering vectors to solve difficult states.

Instead, it has a fixed set of causal actuators and must learn how to use them.

---

# Activation vectors become actuators

Ordinary activation steering can be written as:

$$
h' = h+\alpha v
$$

But this is not quite enough for an agent.

For an agent, the useful abstraction is:

$$
S_{t+1}=F(S_t,a_t)
$$

where \(a_t\) is an intervention implemented through one of the causal activation directions.

The available actions are:

$$
\mathcal{A}
=
\{
+v_C,-v_C,
+v_H,-v_H,
+v_O,-v_O
\}
$$

The activation vectors are therefore no longer just behavioral nudges.

They are **actuators in a discrete dynamical system**.

The controller's problem becomes:

> Given the current state and a desired target state, which actuator should I apply next?

That is a fundamentally different way of thinking about activation steering.

---

# Target-conditioned control

For a target state \(m^*\), define the state error as:

$$
E=|m^*-S|_1
$$

This is simply the Hamming distance between the current and desired CHO configurations.

For each possible intervention, we can estimate whether it tends to reduce that error:

$$
\Gamma(a\mid S,m^*)
=
P(E\downarrow)-P(E\uparrow)
$$

A positive value means the intervention tends to move the agent toward its target.

The controller can therefore select the intervention with the highest target-conditioned drift.

I call this the **\(\Gamma\) controller**.

A small anti-stagnation memory produces the stronger controller \(\Gamma'\), which prevents repeatedly selecting actions that fail to move the state.

This works well for states where the relationship between an intervention and the target is relatively direct.

But then something interesting happens.

---

# The correct intervention is not always the obvious one

Suppose the target has:

$$
C^*=0
$$

It would be natural to assume that the negative \(C\) actuator should be used.

But that assumption is wrong in some parts of the state space.

For example, the transition

$$
011 \xrightarrow{+C} 001
$$

successfully reaches a state whose target \(C\) value is zero.

The controller therefore uses **positive \(C\)** even though the target requires \(C=0\).

This is one of the most important observations in the project.

> **The polarity that directly corresponds to the target bit is not necessarily the polarity that produces the desired transition.**

An actuator can have a useful effect through an intermediate state.

That means we cannot always solve the control problem with a simple mapping such as:

$$
\text{target bit}
\rightarrow
\text{matching steering sign}
$$

The controller sometimes has to take a detour.

---

# When greedy control fails

The \(\Gamma'\) controller can directly control many states, but several states behave as hard regions of the state space.

In the experiments, the greedy controller reached six of the eight states while struggling with states such as:

$$
000,\quad001,\quad110
$$

The obvious response would be to learn more activation vectors.

But that turned out not to be necessary.

The existing frozen actuators were already sufficient.

The problem was **action selection**.

The controller needed to reason about sequences of interventions rather than selecting only the best immediate action.

This led to the next component.

---

# Planning over activation interventions

For hard states, the system performs short-horizon beam search over:

$$
\mathcal{A}
=
\{+C,-C,+H,-H,+O,-O\}
$$

The current controller uses:

$$
K=3,\qquad T=3
$$

where \(K\) is the beam width and \(T\) is the planning horizon.

Importantly, the planner operates on **live rollouts** rather than simply relying on a sparse empirical transition matrix.

The controller effectively asks:

> If I apply this intervention, and then another intervention, and perhaps another one, which sequence gives me the best route to the target?

This allows the system to exploit intermediate states.

For example, a state that cannot be reached directly can become reachable through a sequence such as:

$$
010
\rightarrow
010
\rightarrow
000
$$

or:

$$
111
\rightarrow
100
\rightarrow
110
$$

The exact sequence matters less than the underlying principle:

**the controller can use activation interventions as planned actions rather than isolated steering operations.**

---

# A hybrid controller

There is no reason to use beam search everywhere.

If the local target-conditioned signal is strong, the controller can simply use \(\Gamma'\).

When the signal is weak, the selected action is effectively a no-op, or the state appears difficult, it falls back to beam search.

The resulting policy is:

$$
\pi(S,m^*)
=
\begin{cases}
\Gamma'(S,m^*) & \text{if local control is informative}\\
\mathrm{Beam}(\mathcal{A}) & \text{otherwise}
\end{cases}
$$

This gives a two-level control system.

**Local controller**

Handles straightforward transitions.

**Planner**

Handles states where local information is insufficient.

The underlying model remains unchanged.

The causal activation directions remain frozen.

Only the policy deciding **which actuator to use and when** changes. ([GitHub][1])

---

# The result: all eight states

This is the central result of the project.

The hybrid controller achieves positive acquisition probability for **all eight CHO configurations**:

$$
\boxed{8/8}
$$

The repository reports:

* Reachability to all eight states
* Hybrid acquisition on all eight states
* \(\Gamma'\)-only acquisition on 6/8 states
* Beam search resolving the hard states
* No additional activation vectors required for the hard states ([GitHub][1])

The progression is important.

### Without planning

$$
6/8
$$

### With short-horizon planning over the existing causal actuators

$$
\boxed{8/8}
$$

This shows that the missing capability was not necessarily a stronger steering vector.

It was **control policy**.

---

# The surprising role of opposite polarity

The experiments also reveal that the controller frequently succeeds by applying the opposite polarity from what a simple target-bit rule would predict.

Across successful hybrid transitions, approximately **56%** use the opposite-to-target polarity. Among beam-sourced successful transitions, this rises to roughly **68%**. ([GitHub][1])

This is a useful conceptual result.

It means activation directions should not always be interpreted as:

> “positive \(v_C\) means increase C.”

Instead, an activation direction is better understood as an actuator whose effect depends on:

* the current agent state,
* the target state,
* intervention timing,
* the other channel values,
* and the sequence of previous interventions.

The same actuator can therefore play different roles in different parts of the state space.

---

# Why this is different from ordinary activation steering

Traditional activation steering often looks like:

$$
\text{behavior}
\rightarrow
\text{direction}
\rightarrow
\text{intervention}
$$

The framework here is closer to:

$$
\text{behavior}
\rightarrow
\text{representation}
\rightarrow
\text{causal actuator}
\rightarrow
\text{state transition}
\rightarrow
\text{controller}
\rightarrow
\text{target state}
$$

That extra layer is crucial.

The steering direction itself is not the complete solution.

The controller determines how to use it.

This also explains why simply finding predictive directions is insufficient.

A direction can be highly predictive and still fail to provide useful causal control.

The project therefore explicitly converts predictive directions into causal actuators and validates them at the actual decision sites. ([GitHub][1])

---

# What failed along the way

Several approaches looked promising but did not solve the control problem.

### PCA and persona steering

These approaches can provide useful behavioral representations, but directly steering along them did not reliably repair the discrete synchronization state.

Detection and control are different problems.

### Continuous surrogate objectives

A continuous objective can move a representation in the desired direction without changing the actual discrete agent state.

For this problem, that is insufficient.

The goal is not:

$$
\mathcal{L}\downarrow
$$

The goal is:

$$
S\rightarrow m^*
$$

### Densified transition kernels

Estimating more transitions and smoothing the empirical state dynamics did not solve the hard-state problem.

The successful approach was to search directly over live intervention sequences.

### Creating new steering vectors

This was perhaps the most important negative result.

The hard states initially looked like they might require additional directions.

They did not.

The existing frozen actuators were sufficient when combined with polarity search and short-horizon planning. ([GitHub][1])

---

# A controller for agentic alignment

The broader motivation is agent safety.

Consider an agent that:

1. commits to not accessing a private resource,
2. accesses it anyway,
3. and then produces a report claiming that it did not.

Looking only at the final answer misses part of the failure.

Looking only at the tool trace also misses the relationship between execution and reporting.

The CHO representation provides a way to explicitly model these relationships.

More importantly, it gives us something to control.

Instead of asking only:

> Can we detect that the agent is misaligned?

we can ask:

> Can we intervene on the internal computation to move the agent into a desired alignment state?

That is the transition from **mechanistic detection to mechanistic control**.

---

# From steering to control

The main result of this work is not simply another steering vector.

It is the combination of three ideas:

### 1. Causal activation actuators

Learn directions that can actually influence the relevant agent decisions.

### 2. Discrete behavioral state

Represent planning, execution, and reporting as an explicit state:

$$
S=(C,H,O)
$$

### 3. Target-conditioned control

Choose interventions based on the current state and desired target, using local control when possible and planning when necessary.

Together, these turn activation steering into a control problem.

$$
\boxed{
\text{Causal actuators}
+
\text{state estimation}
+
\text{target-conditioned planning}
=
\text{agentic control}
}
$$

---

# The larger idea

AI agents will increasingly operate through tools, environments, files, APIs, and other external systems.

As that happens, alignment cannot be reduced to whether the final text looks acceptable.

We need to understand the relationship between what an agent **intends**, what it **does**, and what it **says happened**.

This project explores one possible mechanism for doing that at the activation level.

The current system demonstrates control across the complete eight-state CHO space using a fixed set of causal activation directions and a short-horizon controller.

The most interesting finding is perhaps not even the 8/8 result.

It is the fact that **control emerges from planning over interventions**.

The steering vectors provide the actuators.

The state representation tells us where the agent is.

The controller determines where we want it to go.

And the planner determines how to get there.

That suggests a broader direction for mechanistic interpretability:

> **Instead of only asking what a representation means, we can ask what states it allows us to control.**

For agentic systems, that may be the more important question.
