---
title: 'Jev 1.13 on the LLM-AggreFact Benchmark'
date: 2026-09-24
permalink: /posts/2026/09/jev-llm-aggrefact/
tags:
  - fact-checking
  - LLMs
  - benchmarks
---

*29,320 claims, 11 datasets, $1.29.*

---

## TL;DR

I ran TypeSafe's "System One" model, **Jev 1.13**, against the full test split of **LLM-AggreFact**, the standard public benchmark for grounded fact-checking. One `noul` (yes/no) question per claim, threshold 0.5, no tuning, no per-dataset prompts.

| | |
|---|---|
| **Macro-average balanced accuracy** | **78.7** (95% bootstrap CI: 77.7–79.8) |
| Current public SOTA (Bespoke-Minicheck-7B) | 77.4 |
| Best LLM on the leaderboard (Claude-3.5 Sonnet) | 77.2 |
| Examples evaluated | 29,320 (full test set, all 11 datasets) |
| Total cost | **$1.29** |

That number would put Jev at the top of the [LLM-AggreFact leaderboard](https://llm-aggrefact.github.io/), ahead of a 7B model purpose-built for this exact task and ahead of every frontier LLM that has been submitted.

It also comes with caveats that matter more than the headline. Those are in the second half.

---

## What Jev actually is

TypeSafe AI released Jev on September 15, 2026, describing it as the first **System One Model**. The central claim is that autoregressive text generation is the wrong interface for automation workloads.

**Jev does not generate text.** You send it a `state` (a string, a JSON object, or an array of strings) and a set of `questions`, and it returns a typed answer per question with a probability distribution. Three question types:

- **`noul`** — binary true/false. Returns a single probability.
- **`choice`** — pick one of up to 255 options. Returns the winner, a probability per option, and a confidence score derived from the shape of the distribution.
- **`score`** — place the subject on an ordered scale of 2–10 levels you define. Returns a probability-weighted average, the per-level distribution, and a confidence.

Every question in a request is evaluated **in parallel against the same state**, so multiple questions cost one round trip. There is no autoregressive decoding.

The training method is **Reinforcement Learning for Calibrated Decisions (RLCD)**, described as optimizing for calibrated probabilities rather than human preference (RLHF) or verifiable rewards (RLVR).

Published pricing: **$0.042 per million input tokens. Output tokens free.**

### The hype

A search for "jev" on GitHub returns over 11,000 repositories, spanning content moderation, trading agents, email triage, and browser automation. Academic interest is also growing: as of late September 2026, over 25 papers on arXiv reference Jev or the System One paradigm.

TypeSafe's own numbers are **193.6x faster, 444.6x cheaper** on their internal workflow evals — with the caveat that those workflows were built by their own team and reference answers are averages of two competitor models.

---

## Why fact-checking, and why LLM-AggreFact

Grounded fact-checking is a favorable test case for Jev — worth being upfront about:

1. **It's natively a `noul` question.** "Is this claim supported by this document?" is a binary proposition.
2. **It's the canonical guardrail workload.** If Jev's stated use case is verification and guardrailing, then hallucination detection on grounding documents is the use case, not a proxy for it.
3. **Volume is the point.** As the LLM-AggreFact authors put it: "A response from an LLM might consist of many sentences. To identify and localize errors, a fact-checker needs to be called many times. If we use GPT-4 as the fact-checker, we can easily spend >10x more to verify the response than we did to produce it in the first place!" A 100x cost reduction changes what you can afford to check.
4. **There's a real leaderboard with 39 models on it**, including frontier LLMs and specialized models down to 0.4B parameters, so "good" has an unambiguous meaning.

### The MiniCheck paper

LLM-AggreFact was introduced in **"MiniCheck: Efficient Fact-Checking of LLMs on Grounding Documents"** (Liyan Tang, Philippe Laban, Greg Durrett — EMNLP 2024, pages 8818–8847). The benchmark aggregates human-annotated (document, claim, label) tuples across diverse sources — Wikipedia paragraphs, news, interviews, and web text — spanning domains including dialogue, science, and healthcare. Full methodology, including dataset construction and the synthetic data pipeline used to train the MiniCheck models, is described in the paper.

The paper covers **10** datasets (~12,949 test examples). The live leaderboard added **RAGTruth** afterwards, bringing it to **11 datasets and 29,320 test examples** — RAGTruth alone accounts for 16,371 examples, more than half the benchmark. This run uses the 11-dataset version.

The four source categories:

| Type | Datasets |
|---|---|
| Summarization | AggreFact-CNN/XSum, TofuEval-MediaS/MeetB, RAGTruth |
| Retrieval-augmented generation | ClaimVerify, LFQA, ExpertQA, RAGTruth |
| Post-hoc grounding | ExpertQA, REVEAL, FactCheck-GPT |
| Human-written claims | WiCE |

### The metric

Balanced accuracy: `BAcc = ½ (TP/(TP+FN) + TN/(TN+FP))`.

The leaderboard "Average" column is the **unweighted macro average across the 11 datasets** — AggreFact-CNN's 558 examples count exactly as much as RAGTruth's 16,371.

---

## The harness

I streamed the `lytang/LLM-AggreFact` test split from HuggingFace, issued one Decisions API call per example, and wrote a per-call CSV (input tokens, output tokens, API-reported cost, prediction, gold label).

The single question, for every one of the 29,320 examples:

```python
{
  "model": "typesafe/jev-1.13",
  "state": {"context": doc, "claim": claim},
  "questions": {
    "supported": {
      "type": "noul",
      "instructions": (
        "Is every part of `claim` fully substantiated by `context`? "
        "All information in the claim must be supported by the context."
      ),
      "criteria": {
        "true":  "The claim is fully entailed or implied by the context",
        "false": "The claim is not fully substantiated by the context",
      },
    }
  }
}
```

This is modeled on the paper's zero-shot LLM prompt (Table 23):

> "Determine whether the provided claim is consistent with the corresponding document. Consistency in this context implies that all information presented in the claim is substantiated by the document. If not, it should be considered inconsistent."

Protocol matches the leaderboard: no claim decomposition, threshold fixed at 0.5, full test set, macro average across datasets. Jev does not accept a raw prompt, so the `instructions`/`criteria` wording is mine, modeled on the paper's.

---

## Results

### Leaderboard

| Model | Size | Average | CNN | XSum | MediaS | MeetB | WiCE | REVEAL | Claim Verify | Fact Check | Expert QA | LFQA | RAG Truth |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **Jev 1.13 (this run)** | – | **78.7** | 68.1 | 75.8 | 75.5 | 83.8 | **83.5** | **89.6** | 77.5 | 77.0 | 59.6 | **88.6** | **86.9** |
| Bespoke-Minicheck-7B | 7B | 77.4 | 65.5 | **77.8** | **76.0** | 78.3 | 83.0 | 88.0 | 75.3 | 77.7 | 59.2 | 86.7 | 84.0 |
| Claude-3.5 Sonnet | – | 77.2 | 67.6 | 75.1 | 73.4 | **84.6** | 77.7 | 89.1 | 71.4 | 77.8 | **60.9** | 85.6 | 86.1 |
| Granite Guardian 3.3 | 8B | 76.5 | 67.0 | 74.9 | 74.0 | 78.6 | 76.6 | **89.6** | 75.9 | 76.1 | 59.6 | 86.9 | 82.2 |
| Mistral-Large 2 | 123B | 76.5 | 64.8 | 74.7 | 69.6 | 84.2 | 80.3 | 87.7 | 71.8 | 74.5 | 60.8 | 87.0 | 85.9 |
| gpt-4o-2024-05-13 | – | 75.9 | 68.1 | 76.8 | 71.4 | 79.8 | 78.5 | 86.5 | 69.0 | 77.5 | 59.6 | 83.6 | 84.3 |
| FactCG-DeBERTa-L | 0.4B | 75.6 | **70.1** | 73.9 | 72.3 | 74.3 | 74.2 | 88.4 | **78.5** | 72.1 | 59.1 | 86.7 | 82.3 |
| Qwen2.5-72B-Instruct | 72B | 75.6 | 63.6 | 73.0 | 71.9 | 80.4 | 80.2 | 88.9 | 70.0 | 77.0 | 60.1 | 84.3 | 81.9 |
| MiniCheck-Flan-T5-L | 0.8B | 75.0 | 69.9 | 74.3 | 73.6 | 77.3 | 72.2 | 86.2 | 74.6 | 74.7 | 59.0 | 85.2 | 78.0 |
| Llama-3.3-70B-Instruct | 70B | 74.5 | 68.7 | 74.7 | 69.5 | 78.4 | 76.6 | 85.5 | 67.4 | 78.5 | 58.3 | 79.8 | 82.6 |
| Llama-3.1-405B-Instruct | 405B | 74.4 | 64.8 | 75.1 | 68.6 | 81.2 | 71.8 | 86.4 | 67.5 | **79.4** | 58.5 | 81.9 | 82.9 |
| QwQ-32B-Preview | 32B | 71.8 | 57.0 | 71.6 | 69.3 | 78.5 | 72.3 | 86.2 | 67.7 | 75.6 | 60.0 | 78.9 | 72.4 |

![Jev 1.13 vs. leaderboard models on LLM-AggreFact]({{ site.url }}{{ site.baseurl }}/images/jev_benchmark_chart.png)

**+1.3 over SOTA.** A 2,000-resample bootstrap over the per-example predictions puts the 95% CI at **[77.7, 79.8]**, with P(macro > 77.4) = 0.994.

Head to head against Bespoke-Minicheck-7B, Jev wins **8 of 11 datasets**:

| Dataset | Jev | Bespoke-7B | Δ |
|---|---|---|---|
| TofuEval-MeetB | 83.8 | 78.3 | **+5.5** |
| RAGTruth | 86.9 | 84.0 | **+2.9** |
| AggreFact-CNN | 68.1 | 65.5 | **+2.6** |
| ClaimVerify | 77.5 | 75.3 | **+2.2** |
| LFQA | 88.6 | 86.7 | **+1.9** |
| REVEAL | 89.6 | 88.0 | **+1.6** |
| WiCE | 83.5 | 83.0 | +0.5 |
| ExpertQA | 59.6 | 59.2 | +0.4 |
| TofuEval-MediaS | 75.5 | 76.0 | −0.5 |
| FactCheck-GPT | 77.0 | 77.7 | −0.7 |
| AggreFact-XSum | 75.8 | 77.8 | −2.0 |

Against the *per-dataset best* across all leaderboard models, Jev takes the top spot on 4 of 11 (WiCE, REVEAL tied, LFQA, RAGTruth) and is never more than 2.4 points off the best.

### Where it's strong and where it isn't

The shape of the errors is more informative than the average.

| Dataset | n | BAcc | TPR | TNR | % positive | avg input tokens |
|---|---|---|---|---|---|---|
| AggreFact-CNN | 558 | 68.1 | 95.8 | **40.4** | 89.8 | 1,063 |
| AggreFact-XSum | 558 | 75.8 | 66.3 | 85.3 | 51.1 | 794 |
| ClaimVerify | 1,088 | 77.5 | 87.8 | 67.2 | 72.5 | 2,617 |
| ExpertQA | 3,702 | **59.6** | 49.5 | 69.6 | 80.3 | 985 |
| FactCheck-GPT | 1,566 | 77.0 | 61.2 | 92.9 | 24.0 | 483 |
| LFQA | 1,911 | 88.6 | 85.8 | 91.4 | 58.7 | 787 |
| RAGTruth | 16,371 | 86.9 | 85.1 | 88.7 | 92.2 | 1,042 |
| REVEAL | 1,710 | **89.6** | 89.5 | 89.8 | 23.4 | 489 |
| TofuEval-MediaS | 726 | 75.5 | 92.8 | 58.1 | 76.3 | 1,406 |
| TofuEval-MeetB | 772 | 83.8 | 92.9 | 74.7 | 80.6 | 1,406 |
| WiCE | 358 | 83.5 | 77.5 | 89.5 | 31.0 | 2,413 |
| **Pooled (all 29,320)** | 29,320 | 82.9 | 80.6 | 85.2 | 78.6 | 1,046 |

Two patterns:

**Jev leans "supported."** TPR exceeds TNR on 6 of 11 datasets, dramatically so on AggreFact-CNN (95.8 vs 40.4) and TofuEval-MediaS (92.8 vs 58.1). On AggreFact-CNN it catches 96% of supported claims and misses 60% of unsupported ones. For hallucination detection, that specific failure mode is the consequential one; since Jev returns a probability, moving the threshold below 0.5 is the straightforward fix.

**ExpertQA is hard for everybody.** Jev gets 59.6; the best model on the leaderboard gets 60.9; the worst gets 58.3. A 2.6-point spread across 39 models. That column is close to noise.

*Note: the pooled BAcc over all 29,320 examples is 82.9, not 78.7. The difference is the macro-averaging: the two AggreFact sets contribute 558 examples each but 1/11th of the score, while RAGTruth's 16,371 examples also get 1/11th. 78.7 is reported here because that is the leaderboard's metric.*

---

## Cost

$0.042 per million input tokens, output free. For 30,659,886 input tokens and 586,400 output tokens:

**Total: $1.287715.** For the entire benchmark.

That's **$0.0439 per 1,000 claims**, or about **23,000 claims per dollar**. Average 1,046 input tokens per call.

| System | Cost for 29,320 claims | $ / 1k claims | vs Jev |
|---|---|---|---|
| MiniCheck-Flan-T5-L (self-hosted, A6000 @ $0.80/hr) | ~$0.54 | $0.019 | **0.42x** |
| **Jev 1.13 (API)** | **$1.29** | **$0.044** | **1x** |
| gpt-4o (current pricing, my token counts) | ~$83 | $2.81 | 64x |
| Claude 3.5 Sonnet (my token counts) | ~$101 | $3.44 | 78x |
| gpt-4o (May 2024 launch pricing) | ~$162 | $5.53 | 126x |
| GPT-4 (paper Table 4, scaled to 29,320) | ~$242 | $8.26 | **188x** |

**Against LLMs, the claim holds.** 64–188x cheaper than using a frontier chat model as a fact-checker. The paper's argument — that verifying a response with GPT-4 can cost 10x more than generating it — mostly evaporates at $0.044 per 1,000 claims.

**Against a self-hosted small model, it does not.** MiniCheck-Flan-T5-L is *2.4x cheaper per example* than Jev if you already have a GPU, at 75.0 versus 78.7 BAcc. FactCG-DeBERTa-L is 0.4B parameters and scores 75.6.

| Option | Pros | Cons |
|---|---|---|
| Self-hosted small model (e.g., MiniCheck-FT5, FactCG-DeBERTa) | Lowest marginal cost at scale; data stays on-prem; no vendor dependency | GPU required; ops overhead; 3–4 BAcc points below Jev on this benchmark |
| Task-specific fine-tuned model | Highest accuracy ceiling on your distribution | Requires labeled data; significant upfront training cost; ongoing maintenance |
| Jev API | No infrastructure; strong zero-shot accuracy; calibrated probability output | Higher per-call cost than self-hosted; opaque model; vendor dependency |
| Frontier LLM (GPT-4o, Claude) | Interpretable chain-of-thought; general capability | 64–188x more expensive per claim for this task |

The honest framing of Jev's cost position on this task:

> +3.7 BAcc over the best sub-1B open model, +1.3 over the 7B SOTA, at 2.4x the marginal cost of the former, with zero infrastructure, no model to download, no GPU to rent, no vLLM to configure.

For most teams that is a better trade than $0.54. For anyone running fact-checks at real volume with a GPU already in the rack, it isn't. Notably, the LLM-AggreFact authors put model size in the leaderboard as a first-class column; Jev's size is undisclosed, which means it can't be placed on that axis at all.

---

## Limitations

### 1. No interpretability

Jev returns a number. There is no reasoning trace, no cited span, no explanation — the model produces no text output. When Jev assigns a claim a probability of 0.31, there is no way to determine whether it identified a genuine contradiction, was confused by a date, or pattern-matched on something irrelevant.

The MiniCheck authors describe the same limitation of their own models:

> "Like many other specialized fact-checking models, our models do not reveal their internal decision-making processes, making it challenging to localize errors to particular mismatched spans of a claim or document."

One partial mitigation: since Jev evaluates all questions in a request in parallel, asking one `noul` per atomic fact localizes *which* fact failed at no latency cost and near-zero marginal token cost — though this identifies the location of failure, not the reason for it. For a production guardrail, this mode is worth considering.

### 2. Contamination cannot be ruled out

LLM-AggreFact ships a `contamination_identifier` field because this is a known hazard. The dataset has been public on HuggingFace since 2024; Jev 1.13 was trained in 2026; TypeSafe does not disclose their training data. This applies equally to every frontier model on the leaderboard, not just Jev.

---

## What I actually think

The headline result holds: **Jev 1.13 scores 78.7 macro BAcc on the full LLM-AggreFact test set, the best published number on that leaderboard, for $1.29 and with no GPU and no fine-tuning.** No threshold tuning, no subset sampling, no prompt sweeping. Anyone with an OpenRouter key can replicate it.

The more useful framing is the shape of the trade:

- Against **frontier LLMs as fact-checkers**, Jev is a clear win — better accuracy and 64–188x cheaper. The MiniCheck paper's core complaint, that verification costs more than generation, does not hold at $0.044 per 1,000 claims.
- Against **purpose-built small open models**, the picture is more nuanced. +3.7 BAcc over MiniCheck-Flan-T5-L at 2.4x the marginal cost, in exchange for zero infrastructure. The right answer depends on volume and whether you already own GPUs.
- The **interpretability gap is the defining constraint.** You are buying a number. If you need a reason, you need something else in the loop.

The more interesting implication may be economic: at $0.044 per 1,000 claims, verifying every sentence of every LLM response in production becomes operationally trivial. Whether that changes what teams actually deploy is an open question.

I'm going to re-run this with probabilities logged and a per-dataset threshold sweep. That would actually test the calibration claim RLCD is making, which is the more important property to evaluate.

---

## Sources

**Jev / TypeSafe**
- [Introducing System One Models & Jev](https://typesafe.ai/blog/introducing-system-one-models-and-jev) — TypeSafe AI, Sep 15 2026
- [How to Use Jev: Moderation with the Jev API in TypeScript](https://openrouter.ai/blog/tutorials/how-to-use-jev/) — Kenny Rogers, OpenRouter, Sep 23 2026
- [Building a Harness with Jev](https://www.langchain.com/blog/building-a-harness-with-jev) — Sydney Runkle & Hunter Lovell, LangChain, Sep 17 2026
- [TypeSafe workflow evals](https://evals.typesafe.ai/) · [Jev 1.13 jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13) · [Confidence docs](https://docs.typesafe.ai/confidence)

**LLM-AggreFact / MiniCheck**
- Liyan Tang, Philippe Laban, Greg Durrett. [MiniCheck: Efficient Fact-Checking of LLMs on Grounding Documents](https://aclanthology.org/2024.emnlp-main.499.pdf). EMNLP 2024, pp. 8818–8847.
- [LLM-AggreFact leaderboard](https://llm-aggrefact.github.io/) (39 models, 11 datasets; accessed Sep 2026)
- [LLM-AggreFact blog post](https://llm-aggrefact.github.io/blog)
- [MiniCheck GitHub](https://github.com/Liyan06/MiniCheck) · [benchmark_evaluation_demo.ipynb](https://github.com/Liyan06/MiniCheck/blob/main/benchmark_evaluation_demo.ipynb)
- [lytang/LLM-AggreFact on HuggingFace](https://huggingface.co/datasets/lytang/LLM-AggreFact) · [Bespoke-Minicheck-7B](https://huggingface.co/bespokelabs/Bespoke-Minicheck-7B)

```bibtex
@InProceedings{tang-etal-2024-minicheck,
  title = {MiniCheck: Efficient Fact-Checking of LLMs on Grounding Documents},
  author = {Liyan Tang and Philippe Laban and Greg Durrett},
  booktitle = {Proceedings of the 2024 Conference on Empirical Methods in Natural Language Processing},
  year = {2024},
  publisher = {Association for Computational Linguistics},
  url = {https://arxiv.org/pdf/2404.10774}
}
```