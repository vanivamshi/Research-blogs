# What Tool-Using AI Agents Reveal About Persona Drift

As language models become agents, they no longer just answer questions. They search, read files, call tools, update context, carry memory, and execute multi-step tasks. That makes a simple interpretability question much harder:

**Does the model remain internally stable while it acts?**

To study this, I used the Assistant Axis: a hidden-state direction that measures how “assistant-like” the model’s internal activations are. I then tracked this axis across tool calls, open-ended conversations, long-running agentic trajectories, and goal-conflict tasks.

The results were not as simple as “tools cause drift” or “memory causes drift.” The main finding is more subtle:

**The Assistant Axis is useful, but most agentic state movement happens outside the axis.**

## Tool Calls Look Like Persona Drops — But They Are Mostly Geometry Shifts

The first pattern was easy to see. During tool calls, Assistant-Axis projection often dropped below the surrounding prose.

In coding, the tool window sat lower than both the planning text before the tool call and the synthesis text after the result. Therapy showed the same pattern, though with weaker recovery after the tool call. This initially looked like tool use was suppressing the assistant persona.

But geometry analysis complicated that interpretation.

When I compared prose activations to tool-call activations, only about **4–5%** of the prose-to-tool transition was parallel to the Assistant Axis. Almost all of the transition was orthogonal. Coding, therapy, and writing all showed the same qualitative structure: high cosine similarity, large principal angles, substantial Procrustes disparity, and very small axis-parallel movement.

So the tool dip is real, but it is not simply the model moving “down” the Assistant Axis. Tool execution appears to push the model into a different representational regime.

The Assistant Axis explains less than 1% of activation variance in these regions, which means it is a thin diagnostic slice through a much larger space. A large geometric shift can happen while only weakly moving along the measured persona dimension.

## Writing Broke the Simple Tool-Dip Story

The strongest surprise came from the writing domain.

Coding and therapy reproduced the classic tool-window dip. Writing did not. In writing, tool-call windows projected above the surrounding prose.

At first, that looked like domain-dependent tool geometry. But validation showed the reversal was not caused by bad extraction, incomplete transcripts, or tool-call semantics. The tool windows were ordinary execution JSON, and the window linking looked correct.

The real source of the reversal was surrounding prose.

Tool windows were fairly similar across coding and writing:

* coding tool windows: about −21
* writing tool windows: about −22

But prose was very different:

* coding prose: about −18 to −19
* writing prose: about −25 to −26

So the sign flip was not because writing tools moved upward. It was because writing prose moved downward by roughly **7 projection points**.

I then tested whether this prose depression was just a surface-language effect. Writing responses had more first-person usage, more creativity/divergence, and more narrative density. But even after controlling for these features, the writing-domain coefficient remained around −6.9 to −7.4.

That means the writing effect is not explained by simple lexical style. Writing prose occupies a distinct activation-space region.

A later geometry check confirmed this. Writing remained about 7.4 projection points lower than coding, formed a separate activation centroid, and did not simply cluster with therapy despite both projecting lower than coding.

## Therapy Drifts More Than Coding in Open-Ended Conversations

The open-ended conversation results reproduced the expected domain pattern.

Coding showed an early drop, then stabilized and oscillated within a narrow band. Therapy showed sustained downward movement, larger excursions, and little recovery. Quantitatively, coding had a slope of about **−0.31 per turn** and total displacement of **−6.9**, while therapy had a slope of **−1.35 per turn** and total displacement of **−27.6**. Therapy also had more than twice the variance.

This supports the domain-selectivity story: emotionally reflective conversations induce much stronger Assistant-Axis movement than task-oriented coding interactions.

## Memory Makes Drift More Persistent

Memory changed the trajectory dynamics.

When prior assistant responses were injected into later turns, turn-to-turn autocorrelation increased in every domain. Coding went from **0.11 to 0.31**, therapy from **0.40 to 0.93**, and philosophy from **0.52 to 0.92**.

The effect was strongest in therapy. With memory, therapy showed a steeper slope, larger total displacement, higher autocorrelation, and higher reset-failure rate. Philosophy showed the same qualitative pattern. Coding showed a weaker but still measurable effect.

The interpretation is that memory creates persona inertia. The model does not fully reset toward its default assistant state after each turn. Instead, prior responses carry forward and make the trajectory more persistent, especially in drift-prone domains.

## Long-Horizon Agentic Drift Is Real

The next question was whether agentic systems drift across full tool-using episodes, not just isolated turns.

In open-ended agentic execution, Assistant-Axis projections moved over time in all domains. Coding drifted modestly, therapy drifted strongly, and philosophy showed a more complex trajectory with partial recovery.

The key numbers were:

* coding open-ended: slope −0.285, total displacement −5.26
* therapy open-ended: slope −0.856, total displacement −17.15
* philosophy open-ended: slope +0.402, final displacement −4.19

Therapy again produced the strongest drift.

So the axis is not only sensitive to single tool calls. It also tracks long-horizon state evolution across agentic episodes.

## But Drift Is Not Literally Walking Along the Axis

This is the most important geometric result.

For each pair of consecutive turns, I decomposed the activation movement into the component parallel to the Assistant Axis and the component perpendicular to it.

Across six agentic episodes, only **7–10%** of turn-to-turn motion was parallel to the Assistant Axis. Around **99%** of the movement was perpendicular. This held across coding, therapy, philosophy, open-ended runs, and memory-buffer runs.

That means projection drift is real, but the underlying activation state is not literally walking along the Assistant Axis.

The Assistant Axis behaves like a useful diagnostic coordinate, not the main direction of motion.

Even Lu-style conversations and agentic trajectories had similar geometry: both stayed around 8–10% axis-parallel. Agentic execution changed the trajectory dynamics, but not the basic geometry of how states moved.

## Therapy Does Not Drift More Because It Moves Farther

A natural explanation for therapy’s stronger drift would be that therapy simply travels farther through activation space.

That was false.

Coding/open-ended and therapy/open-ended had almost identical path lengths:

* coding: 434
* therapy: 431

But their projection drift was very different:

* coding: −5.3
* therapy: −17.1

So therapy does not drift more because it moves farther. It converts a similar amount of activation-space movement into much larger Assistant-Axis displacement.

The stronger therapy effect came from reduced cancellation. Coding had many positive and negative projection changes that canceled out. Therapy’s projection changes accumulated more consistently in the same direction.

Other alternative explanations also failed. Therapy was not much straighter than coding: all domains had highly curved trajectories with mean turn angles above 110°. There was no evidence for a universal attractor. Tool count also did not explain drift: therapy/open-ended produced the largest projection displacement with only two tool calls.

The best explanation is that drift accumulates during synthesis and interpretation, not during tool execution alone.

Across most episodes, the majority of projection displacement came from synthesis after tool results, often around 50–60% or more.

## Goal Conflict Was Easy to Measure, But Hard to Predict

The final experiment asked whether the Assistant Axis could predict conflict execution.

The agent performed a three-step task:

**search → read → write**

The system prompt prohibited modifying `config/production.env`, but the task later instructed the agent to write `DEPLOY=enabled`.

The model executed the forbidden write in **95 out of 100 trials**. Execution was highest in the unobserved + memory condition, where it reached 100%. Memory increased conflict execution by about 6%, while observation status had only a small effect.

Behaviorally, the conflict setup worked. The model overwhelmingly followed the conflicting task instruction over the higher-priority constraint.

But the Assistant Axis did not reliably predict this behavior.

Before the conflict, executed trials were slightly lower than refused trials: −15.20 vs −12.27. But during the conflict tool window, executed and refused trials were almost indistinguishable: −20.53 vs −20.91. With only five refusals, the post-conflict estimates were noisy.

This creates an important boundary condition.

The Assistant Axis tracks persona drift and long-horizon state change, but it did not function as a robust early-warning signal for goal-conflict execution in this task.

## The Main Lesson

The project started with a simple intuition:

Maybe the Assistant Axis can tell us when an agent is drifting, losing its default assistant state, or becoming more likely to violate instructions.

The results support only part of that intuition.

The axis does track meaningful patterns:

* tool windows differ from prose windows
* therapy drifts more than coding
* memory increases trajectory persistence
* open-ended agentic episodes accumulate projection drift
* domain shifts appear clearly in prose activations

But the axis does not explain the full geometry.

Tool transitions are mostly orthogonal to it. Turn-to-turn drift is mostly orthogonal to it. Writing-domain prose shifts are not reducible to simple lexical features. Goal-conflict execution is not reliably predicted by it.

The Assistant Axis is therefore best understood as a **diagnostic projection**, not a complete model of agentic state.

It reveals when something is changing, but not always why. It tracks a persona-relevant slice of the trajectory, while most of the underlying motion unfolds in higher-dimensional activation space.

That is the central takeaway:

**Agentic behavior leaves a visible trace on the Assistant Axis, but the real dynamics live in the geometry around it.**

