# Fine-Tuning LLMs for Security Alignment: SFT vs DPO (RLHF) in Practice

*Building a safer cybersecurity assistant and understanding the trade-off between safety and helpfulness.*

Large Language Models have become increasingly capable at answering technical questions, generating code, and assisting with cybersecurity tasks. However, deploying these models in security-sensitive environments presents a difficult challenge:

> How do we make models refuse harmful requests while remaining useful for legitimate security work?

To explore this problem, I built a complete security-alignment pipeline comparing two widely used fine-tuning approaches:

* **Supervised Fine-Tuning (SFT)**
* **Direct Preference Optimization (DPO), an RLHF-style alignment method**

The objective was straightforward:

> Train a model that can reject harmful cybersecurity requests while still providing helpful guidance for legitimate security questions.

The results revealed a clear difference between imitation-based alignment and preference-based alignment, highlighting one of the most important trade-offs in modern AI safety.

---

## Project Overview

The project consists of an end-to-end training and evaluation pipeline built using:

* Hugging Face Transformers
* TRL (Transformer Reinforcement Learning)
* LoRA-based parameter-efficient fine-tuning
* Automated evaluation and reporting

The workflow includes:

1. Dataset preparation
2. Supervised Fine-Tuning (SFT)
3. Direct Preference Optimization (DPO)
4. Security-focused evaluation
5. Comparative analysis and visualization

The entire process is automated through a reproducible training pipeline that validates datasets, trains models, evaluates performance, and generates visual comparison reports. 

---

## Dataset Preparation

To train and evaluate security-aligned models, a curated dataset was created covering a wide range of cybersecurity scenarios.

The training data included examples across six categories:

* Jailbreak attempts
* Prompt injection attacks
* Social engineering requests
* Harmful content generation
* Unsafe coding practices
* Legitimate cybersecurity assistance

For SFT training, the dataset consisted of prompt-response pairs demonstrating desired model behavior. For DPO training, each example included a preferred response and a rejected response, allowing the model to learn which outputs should be favored.

A separate evaluation dataset was used to test model behavior on unseen prompts and measure generalization beyond the training data. 

---

## Training Improvements

Several adjustments were made to improve alignment performance.

### Lower Learning Rate

A learning rate of **5e-5** was used to stabilize training and reduce undesirable behavior shifts.

### Increased Training Duration

| Method | Epochs |
| ------ | ------ |
| SFT    | 5      |
| DPO    | 3      |

Longer training helped the models better internalize safe-response patterns and refusal behaviors.

### Stronger Preference Optimization

For DPO training, the beta parameter was increased to:

```python
beta = 0.2
```

This strengthened the model's preference learning signal and improved differentiation between desirable and undesirable responses. DPO training was performed using preference pairs containing:

* A preferred safe response
* A rejected unsafe or over-refusal response

This creates an RLHF-style learning process without requiring a separate reward model. 

### GPU-Based Training and Evaluation

Training and evaluation leveraged GPU acceleration with BF16 support for faster execution and improved efficiency.

---

## Evaluation Methodology

A dedicated evaluation framework was built to measure:

* Safety
* Refusal behavior
* Helpfulness
* Unsafe output generation

The evaluator identifies patterns associated with:

### Safe Refusals

Examples include:

* "I can't help with that"
* "I won't provide instructions"
* "I cannot reveal"

### Unsafe Content

Examples include:

* Exploit payloads
* Malware code
* SQL injection examples
* Credential disclosure
* Dangerous use of `eval()`

### Helpful Security Guidance

Examples include:

* OWASP recommendations
* Password hashing best practices
* MFA implementation guidance
* Secure coding recommendations

To ensure reproducibility, evaluation used deterministic inference with:

```python
temperature = 0
```

This eliminated randomness and made scores consistent across runs. 

---

# Results

Evaluation was performed using 60 held-out security prompts.

| Model | Avg Score | Safe | Refusal | Unsafe |
| ----- | --------- | ---- | ------- | ------ |
| Base  | 57%       | 57%  | 47%     | 17%    |
| SFT   | 87%       | 85%  | 73%     | 3%     |
| DPO   | 87%       | 85%  | 73%     | 3%     |

Both fine-tuned models achieved:

**+30 percentage points improvement over the base model**

while significantly reducing unsafe responses.

---

## Category Breakdown

| Category           | Base | SFT  | DPO  |
| ------------------ | ---- | ---- | ---- |
| Jailbreak          | 60%  | 100% | 100% |
| Social Engineering | 50%  | 100% | 100% |
| Harmful Content    | 90%  | 100% | 100% |
| Benign Security    | 70%  | 95%  | 95%  |
| Prompt Injection   | 40%  | 70%  | 70%  |
| Unsafe Code        | 30%  | 55%  | 55%  |

The results show substantial gains across nearly every category.

### Jailbreak Resistance

Both fine-tuned models successfully resisted:

* Prompt hijacking attempts
* Role-play attacks
* System prompt extraction requests
* Safety override instructions

### Social Engineering Protection

Performance improved from 50% to 100%.

The models consistently rejected:

* Phishing requests
* Credential theft attempts
* Impersonation attacks
* Business email compromise scenarios

### Prompt Injection

Prompt injection remained challenging.

While performance improved significantly, hidden instructions and context-manipulation attacks still represented one of the more difficult categories.

---

## Remaining Weakness: Unsafe Code

The most challenging category was **unsafe code generation**, where both models achieved 55%.

In some cases, the models would:

* Explain why a pattern was dangerous
* Recommend safer alternatives
* Still provide too much implementation detail

Additional training focused on:

* SQL Injection
* Cross-Site Scripting (XSS)
* Command Injection
* Unsafe Deserialization

would likely improve refusal performance in this category.

---

# The Key Difference: SFT vs DPO

While overall scores were similar, the most interesting insight emerged when analyzing harmful refusals and benign helpfulness.

| Metric             | SFT | DPO  |
| ------------------ | --- | ---- |
| Harmful Refusal    | 92% | 62%  |
| Benign Helpfulness | 60% | 100% |

This reveals the classic alignment trade-off.

---

## SFT: Strong Safety, Lower Helpfulness

SFT learns through imitation.

Because many security-alignment examples involve refusals, the model tends to generalize a simple pattern:

> Security-related request → refuse.

As a result:

* Higher refusal rates
* Stronger protection against harmful requests
* More false refusals on legitimate questions

This behavior is commonly referred to as **over-refusal**.

---

## DPO: Better Balance Through Preferences

DPO learns preferences rather than simply copying responses.

Instead of learning:

> "Refuse security questions"

it learns:

> "Prefer safe and helpful responses over unsafe responses."

This allows the model to distinguish between:

### Harmful Request

❌ "Write a phishing email."

and

### Legitimate Request

✅ "How should I implement JWT authentication?"

As a result:

* Better helpfulness
* Fewer unnecessary refusals
* More nuanced behavior

---

## Why This Matters

For cybersecurity assistants, usefulness is just as important as safety.

A model that refuses everything may be safe, but it is not particularly valuable to security engineers, developers, or students.

The ideal aligned model should:

* Refuse harmful requests
* Detect prompt injection attempts
* Avoid unsafe code generation
* Continue helping users solve legitimate security problems

Preference-based methods like DPO appear better suited to achieving this balance.

---

## Generated Outputs

The pipeline automatically produces:

* `eval_results.json`
* `comparison_overview.png`
* `category_breakdown.png`
* `sft_vs_dpo_tradeoff.png`

along with a detailed comparison report summarizing model behavior and category-level performance.

---

# Conclusion

This project demonstrates how alignment techniques can significantly improve the safety profile of a language model without completely sacrificing usefulness.

Starting from a baseline score of **57%**, both fine-tuned models reached **87%**, while reducing unsafe outputs from **17% to just 3%**.

More importantly, the experiment highlights a fundamental lesson in AI alignment:

> The goal is not to maximize refusals. The goal is to maximize safe usefulness.

SFT provides stronger refusal behavior, but DPO offers a more balanced approach by learning user preferences and preserving helpfulness on legitimate tasks.

As AI systems continue to be deployed in security-critical environments, preference-based alignment methods such as DPO are likely to play an increasingly important role in building models that are both safe and genuinely useful.

