# Do Global Markets Have a Hidden Geometry?

*What happens when we stop treating financial markets as prices—and start treating them as evolving information networks?*

---

## Markets don't just move. They communicate.

When the S&P 500 moves, Europe reacts.

When Europe closes, Asia inherits information.

Volatility spreads across continents.

Liquidity shocks propagate.

We've known this for decades.

But there's a deeper question:

> **What does the entire system of information flow look like?**

Not one edge.

Not one correlation.

Not one forecasting model.

The whole system.

---

## Most research studies individual relationships

Finance has many tools:

* Correlations
* Granger causality
* Transfer entropy
* Spillover indices
* VARs

Each asks whether one market helps explain another.

That's valuable.

But imagine trying to understand the Internet by looking at one cable at a time.

The interesting object isn't a single connection.

It's the network.

---

## A different perspective

Instead of studying individual predictive relationships, I constructed a **directed graph** every week.

Each node is a market.

Each edge measures how much one market improves the prediction of another using incremental out-of-sample ( \Delta R^2 ).

So every week produces a different graph.

Now the question changes.

Instead of asking

> "Does market A predict market B?"

we ask

> "How does the entire predictive-information network evolve over time?"

---

## Graphs become points

Graphs are complicated.

Comparing hundreds of them directly is difficult.

So I summarized each graph using five interpretable structural properties:

* entropy
* effective rank
* dominant singular value
* sparsity
* temporal concentration of predictive information

Each graph becomes one point in a five-dimensional space.

Now every week corresponds to a location.

As markets evolve, those points trace a trajectory.

The predictive-information network becomes a dynamical system.

---

## The surprising part wasn't persistence.

Persistence is expected in finance.

The surprising part was **how smooth the geometry was.**

Neighboring weeks remained close in graph space.

Randomly shuffling time destroyed this continuity.

Even preserving degree sequences or singular spectra couldn't reproduce the observed trajectories.

The organization of the network mattered—not just its summary statistics.

---

## Does the embedding actually preserve graph structure?

This is an important question.

Reducing a graph to five numbers always loses information.

The goal isn't perfect reconstruction.

The goal is preserving meaningful geometry.

To test this, I compared distances between graphs with distances between their embedded representations.

The relationship was strong enough to indicate that nearby graphs remained nearby after embedding, while distant graphs stayed relatively distant.

In other words, the embedding is **faithful but intentionally lossy**.

It captures structural organization without pretending to encode every edge.

---

## Stress changes the geometry

One particularly interesting observation appeared during periods of market stress.

Instead of predictive information spreading across multiple future horizons, it became concentrated near the present.

Information propagation compressed in time.

Rather than flowing gradually, predictive influence became more immediate.

This pattern appeared consistently across stress regimes and survived permutation testing.

---

## This isn't another forecasting paper

The natural question is:

> "Can this predict returns?"

That's actually not the point.

The contribution is different.

The paper treats evolving predictive-information networks as an object worthy of study in their own right.

It's a systems view of financial markets.

Forecasting becomes secondary.

Understanding the organization of information becomes primary.

---

## Why this matters

Modern markets are increasingly interconnected.

Understanding isolated relationships is no longer enough.

Network organization carries information that individual edges cannot.

Thinking geometrically opens new questions:

* How do market regimes appear in graph space?
* When does network topology reorganize?
* Can crises be understood as geometric transitions?
* Do other complex systems exhibit similar dynamics?

These questions extend well beyond finance.

Any evolving network—from biological systems to communication networks—may possess its own hidden geometry.

---

## What's next?

This work currently uses incremental out-of-sample ( \Delta R^2 ) to construct predictive-information graphs.

An obvious next step is testing whether the same geometric behavior appears when edges are estimated using alternative measures such as Granger causality, transfer entropy, or directed information.

If the geometry persists across estimators, that would suggest we're observing a property of the underlying information-flow system rather than a particular statistical measure.

---

## Closing thought

Financial markets are often studied through prices, factors, or forecasts.

This work asks a different question.

> **What if the evolving structure of information flow has its own geometry?**

If so, perhaps the most interesting object in finance isn't the market itself.

It's the shape of the information moving through it.

---
