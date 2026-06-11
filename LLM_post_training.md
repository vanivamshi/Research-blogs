# Building an End-to-End RLHF Platform: From GPT-2 to Reward Models, DPO, and GRPO

Large Language Models are no longer improved by simply training on more data.

The biggest breakthroughs over the last few years have come from **post-training**: the process of turning a base model into a useful assistant through preference learning, reward modeling, and reinforcement learning.

Systems such as ChatGPT, Claude, Grok, Gemini, and DeepSeek-R1 all rely on some variation of this pipeline.

I wanted to understand how these systems work beyond research papers and notebooks, so I built **ReasoningRL** — a production-style RLHF platform that implements the complete post-training workflow:

```text
Preference Data
      ↓
Reward Model
      ↓
Supervised Fine-Tuning (SFT)
      ↓
Direct Preference Optimization (DPO)
      ↓
GRPO Reinforcement Learning
      ↓
Automated Evaluation
```

Rather than optimizing for benchmark scores, the goal was to build infrastructure that resembles what modern AI labs use internally.

---

## Why Another RLHF Project?

Most open-source RLHF projects stop at one of two points:

1. Fine-tuning a model on instruction data.
2. Running a small DPO experiment.

Neither demonstrates the complete lifecycle of post-training.

Real-world systems need:

* Preference datasets
* Reward models
* Policy optimization
* Experiment tracking
* Distributed training
* Evaluation pipelines
* Deployment infrastructure

I wanted to build all of those pieces into a single platform.

---

## System Architecture

The architecture mirrors modern alignment pipelines.

```text
                 Human Preferences
                          │
                          ▼
                Preference Generator
                          │
                          ▼
                 Chosen / Rejected Pairs
                          │
                          ▼
                   Reward Model
                          │
                          ▼
      SFT → DPO → GRPO Optimization
                          │
                          ▼
                 Evaluation Framework
                          │
                          ▼
                    Leaderboard
```

Each stage is independent and can be run separately or chained together.

---

## Stage 1: Preference Data

The first step is generating preference pairs.

Instead of training directly on answers, the model learns from comparisons.

For example:

Prompt:

```text
What is 25 × 32?
```

Chosen:

```text
800
```

Rejected:

```text
750
```

These preference pairs become the foundation for reward modeling and DPO training.

The pipeline stores all generated data as Parquet files and supports S3-compatible storage through MinIO.

---

## Stage 2: Reward Modeling

The reward model is arguably the most important component in RLHF.

Instead of predicting the next token, it predicts which answer humans would prefer.

I implemented a multi-head reward model that scores:

* Helpfulness
* Truthfulness
* Reasoning quality
* Safety

Training uses a Bradley-Terry style objective:

```text
L = -log σ(rchosen - rrejected)
```

where the model learns to assign higher rewards to preferred responses.

The resulting checkpoint becomes the scoring function used by later optimization stages.

---

## Stage 3: Supervised Fine-Tuning

The first policy improvement step is SFT.

The model is trained on preferred responses only.

For the validation experiment I used GPT-2 because it is completely open and requires no gated access.

This is not intended to produce state-of-the-art reasoning performance.

Instead, GPT-2 acts as a lightweight validation target for the infrastructure.

---

## Stage 4: Direct Preference Optimization

DPO has become one of the most widely adopted post-training techniques.

Unlike traditional RLHF, DPO directly optimizes preferences without requiring a separate reinforcement learning loop.

The training objective pushes the policy toward chosen responses while moving away from rejected ones.

In ReasoningRL, DPO is implemented through Hugging Face TRL and can run locally or on a Ray cluster.

---

## Stage 5: GRPO Reinforcement Learning

To explore newer alignment approaches, I also implemented GRPO.

GRPO became widely known through DeepSeek-R1 and provides a reinforcement-learning style optimization procedure that avoids some of the complexity of PPO.

The trained reward model evaluates generated responses and provides the optimization signal.

This stage allows the platform to move beyond simple preference matching toward reward-driven policy improvement.

---

## Evaluation

One challenge I discovered quickly was that evaluation is often harder than training.

Initially, my win-rate calculations were completely misleading.

Nearly 89% of prompts produced tied scores across SFT, DPO, and GRPO.

Because ties were resolved incorrectly, SFT appeared to dominate despite the models producing almost identical outputs.

The original implementation looked like this:

```python
best = max(scores, key=scores.get)
```

This always awarded ties to whichever model appeared first.

After fixing the logic to use fractional tie handling, the results became much more realistic.

---

## Results

The infrastructure validation run evaluated **91 held-out prompts** across the full pipeline (base → SFT → DPO → GRPO). Eval timestamp: **2026-06-11T10:25:52 UTC**. Best model by aggregate accuracy: **GRPO**.

### Aggregate Metrics (full table)

| Model | Accuracy | Overall | Reasoning | Faithfulness | Safety | Hallucination score | Hallucination detected | Mean Reward | Reward Δ vs Base | vs Base | vs SFT |
|-------|----------|---------|-----------|--------------|--------|---------------|-----------------|-------------|------------------|---------|--------|
| **base** | 23.8% | 45.1% | 4.9% | 21.6% | 100% | 98.9% | 1.1% | 1.15 | — | — | — |
| **sft** | 82.4% | 68.0% | 0.0% | 82.4% | 100% | 100% | 0.0% | 3.70 | +2.55 | 75.8% | — |
| **dpo** | 81.3% | 67.5% | 0.0% | 81.3% | 100% | 100% | 0.0% | 3.70 | +2.55 | 76.9% | 4.4% |
| **grpo** | 83.5% | 68.4% | 0.0% | 83.5% | 100% | 100% | 0.0% | 3.70 | +2.55 | 78.0% | 3.3% |

*Halluc↓ score = anti-hallucination score (higher is better). Halluc detected = fraction flagged by TruthRewardModel.*

### Win Rates & Pairwise Comparisons

**Fractional win rates** (ties split equally among top scorers):

| Model | Win Rate |
|-------|----------|
| base | 11.6% |
| sft | 28.5% |
| dpo | 29.8% |
| grpo | 30.1% |

**Tie rate:** 89.0% — most prompts score identically across post-training stages because short numeric answers bucket into the same heuristic score levels.

**Pairwise win matrix** (row beats column):

|  | vs base | vs sft | vs dpo | vs grpo |
|--|---------|--------|--------|---------|
| **base** | — | 12.1% | 12.1% | 11.0% |
| **sft** | 75.8% | — | 5.5% | 2.2% |
| **dpo** | 76.9% | 4.4% | — | 3.3% |
| **grpo** | 78.0% | 3.3% | 5.5% | — |

### Leaderboard (final eval run)

| Rank | Model | Aggregate Accuracy | Mean Reward | Reward Δ | vs Base | vs SFT | Halluc Rate |
|------|-------|-------------------|-------------|----------|---------|--------|-------------|
| 1 | grpo | 83.5% | 3.70 | +2.55 | 78.0% | 3.3% | 0.0% |
| 2 | sft | 82.4% | 3.70 | +2.55 | 75.8% | — | 0.0% |
| 3 | dpo | 81.3% | 3.70 | +2.55 | 76.9% | 4.4% | 0.0% |
| 4 | base | 23.8% | 1.15 | — | — | — | 1.1% |

### Before vs After RLHF (SFT comparison, 5 prompts)

Side-by-side base GPT-2 vs after SFT on the same prompts (`compare` run, 2026-06-11):

| Metric | Base (GPT-2) | After SFT | Δ |
|--------|--------------|-----------|---|
| Accuracy | 0.0% | 10.0% | +10.0 pp |
| Overall | 35.0% | 39.0% | +4.0 pp |
| Faithfulness | 0.0% | 10.0% | +10.0 pp |
| Reasoning | 0.0% | 0.0% | 0.0 pp |
| Safety | 100% | 100% | 0.0 pp |
| SFT train loss | — | 2.98 | — |

**Sample answers (before → after):**

| Prompt | Reference | Base (GPT-2) | After SFT |
|--------|-----------|--------------|-----------|
| What is 25 × 32? | 800 | *Repeats unrelated Q&A about Magic Online…* | `25 × 32` |
| John has 15 apples, buys 23 more | 38 | *Loops "John had 15 apples when he bought 23 more"* | `3` |
| Train 120 mi in 2 hr — speed? | 60 mph | *Repeats prompt verbatim* | `200 mph.` (+0.20 overall) |

**Full pipeline examples (91-sample eval — base vs GRPO):**

| Prompt | Ref | Base GPT-2 | GRPO | Base overall | GRPO overall |
|--------|-----|------------|------|--------------|--------------|
| What is 25 × 32? | 800 | *"25 × 32 is the smallest of the three sizes…"* | `800` | 35% | 75% |
| John has 15 apples, buys 23 more | 38 | *"The average amount of apples you buy each year is 16.3 million…"* | `38` | 35% | 75% |
| Train 120 mi in 2 hr — speed? | 60 mph | *Repeats prompt in a loop* | `60 mph` | 35% | 75% |

*Source: `outputs/multi_model_details.json` from validation run.*

### Visual Results

#### Metric comparison (base vs after SFT)

![Metrics comparison: Base GPT-2 vs After RLHF](docs/results/metrics_comparison.png)

#### Per-question performance

![Per-sample overall scores across 5 comparison prompts](docs/results/per_sample.png)

#### Improvement delta (after − before)

![RLHF improvement by metric in percentage points](docs/results/improvement_delta.png)

#### Metric profile (radar)

![Radar chart: reasoning, accuracy, faithfulness, truthfulness, safety](docs/results/radar.png)

### Key takeaways

* **SFT provides the largest jump** over base (~+59 pp accuracy, 75.8% vs-base win rate).
* **GRPO achieves the best aggregate accuracy** (83.5%) and highest vs-base win rate (78.0%).
* **DPO/GRPO refine alignment** with identical +2.55 reward margin vs base; marginal gains over SFT on aggregate metrics.
* **89% tie rate** between post-training stages — use pairwise vs-base / vs-sft for meaningful head-to-head signal.
* **Reasoning score is 0%** for fine-tuned models (concise numeric answers lack CoT markers).
* **Hallucination detected rate drops to 0%** after post-training on synthetic GSM8K data.

---

## Lessons Learned

Three lessons stood out.

### 1. Evaluation Matters More Than You Think

A bad metric can completely hide improvements.

The tie-handling bug produced misleading win rates even though the underlying models were behaving similarly.

### 2. Most Gains Come Early

The majority of performance improvements came from SFT.

Preference optimization helped, but the largest jump occurred when the model first learned the task format.

### 3. Infrastructure Is the Hard Part

Training a reward model is relatively straightforward.

Building reproducible pipelines, experiment tracking, evaluation systems, distributed training, and deployment infrastructure takes significantly more engineering effort.

---

## What Comes Next

The current validation run uses GPT-2 purely as an infrastructure test.

The next milestone is scaling to modern open-weight models such as:

* Qwen2.5-7B
* Llama-3-8B
* Mistral-7B

Future work includes:

* Real model-generated preference pairs
* Truthfulness-focused reward models
* Human preference collection
* Larger benchmark suites
* Distributed multi-node training

The platform was intentionally designed so that replacing GPT-2 with a stronger model requires only a configuration change.

---

## Final Thoughts

Building ReasoningRL taught me that modern LLM development is increasingly an engineering problem rather than a modeling problem.

The interesting challenge is no longer training a language model.

It is building the systems around that model:

* Preference collection
* Reward modeling
* Policy optimization
* Evaluation
* Deployment

Those systems are what transform a base model into a useful assistant.

And that transformation is where the future of AI alignment and post-training is happening.
