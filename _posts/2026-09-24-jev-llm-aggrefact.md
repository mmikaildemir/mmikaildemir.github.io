---
title: 'I Ran Jev Against the Whole LLM-AggreFact Benchmark. It Came First.'
date: 2026-09-24
permalink: /posts/2026/09/jev-llm-aggrefact/
tags:
  - fact-checking
  - LLMs
  - benchmarks
---

*Roughly five hours of API calls, 29,320 claims, $1.29.*

---

## TL;DR

I ran TypeSafe's new "System One" model, **Jev 1.13**, against the full test split of **LLM-AggreFact**, the standard public benchmark for grounded fact-checking. One `noul` (yes/no) question per claim, threshold 0.5, no tuning, no per-dataset prompts.

| | |
|---|---|
| **Macro-average balanced accuracy** | **78.7** (95% bootstrap CI: 77.7–79.8) |
| Current public SOTA (Bespoke-Minicheck-7B) | 77.4 |
| Best LLM on the leaderboard (Claude-3.5 Sonnet) | 77.2 |
| Examples evaluated | 29,320 (full test set, all 11 datasets) |
| Median latency | **355 ms** |
| Total cost | **$1.29** |

That number would put Jev at the top of the [LLM-AggreFact leaderboard](https://llm-aggrefact.github.io/), ahead of a 7B model purpose-built for this exact task and ahead of every frontier LLM that has been submitted.

It also came with a pile of caveats that I think matter more than the headline. Those are in the second half.

---

## What Jev actually is

On September 15, 2026, TypeSafe AI came out of two years of stealth and [released Jev](https://typesafe.ai/blog/introducing-system-one-models-and-jev), which they call the first **System One Model**. The founder, Diogo Almeida, previously worked on the RLHF methods behind ChatGPT at OpenAI, and the framing of the launch post is that chat was the wrong shape for automation:

> "Models have been superhuman at chat for years, so where is all the automation?"

The pitch is that Jev is not a smaller LLM. It's a different thing. The clearest way to describe it is what it gives up:

**Jev does not generate text.** At all. There is no string output, no chain of thought, no explanation. You send it a `state` (a string, a JSON object, or an array of strings) and a set of `questions`, and it returns a typed answer per question with a probability distribution attached. Three question types:

- **`noul`** — is this statement true? Returns a single probability.
- **`choice`** — pick one of up to 255 options. Returns the winner, a probability per option, and a confidence score derived from the shape of the distribution.
- **`score`** — place the subject on an ordered scale of 2–10 levels you define. Returns a probability-weighted average, the per-level distribution, and a confidence.

Every question in a request is evaluated **in parallel against the same state**, so asking five questions costs one round trip and only the marginal input tokens for the extra question text. There's no autoregressive decoding, which is where the speed comes from.

The training method is something they call **Reinforcement Learning for Calibrated Decisions (RLCD)**, optimizing for "answers with epistemically honest probabilities" rather than human preference (RLHF) or verifiable rewards (RLVR).

Published pricing: **$0.042 per million input tokens. Output tokens free.** For comparison, that's roughly 60–240x below frontier LLM input pricing. Claimed end-to-end latency: 70–500 ms.

### The hype

The launch landed hard. Within about a week:

- **LangChain shipped an integration** and wrote [a whole post about it](https://www.langchain.com/blog/building-a-harness-with-jev), repeating TypeSafe's claim of "up to 200x faster inference and 400x lower cost than comparable LLMs on classification tasks." They built two pieces of middleware around it: `ModelRouterMiddleware` (use Jev to pick which LLM handles a request) and `AutoModeMiddleware` (use Jev to gate risky tool calls before they execute). Their framing of the second one is sharp — the dangerous-action classifier that Claude Code, Codex and Cursor all ship has been locked inside closed-source harnesses, and a cheap performant classifier makes that pattern available to everyone.
- **OpenRouter published a full tutorial**, exposing Jev through their Decisions API at `typesafe/jev-1.13` and a `POST /api/alpha/decisions` endpoint. No TypeSafe account needed.
- People started building: browser-use agents for fractions of a cent, live trading agents, email triage at scale.

TypeSafe's own numbers are extraordinary: **193.6x faster, 444.6x cheaper** on their internal workflow evals. To their credit, they front-load the caveats themselves — the workflows were built by their own model capabilities team, the reference answers are the average of two competitor models, and they explicitly say "we love skeptics, and are skeptics ourselves."

Fine. Let's be skeptics.

---

## Why fact-checking, and why LLM-AggreFact

The interesting thing about the launch was what it *didn't* include. TypeSafe published side-by-side demos, internal workflow evals, Doom, and Wikiracing. The FAQ has an entry titled "How does Jev perform against public benchmarks?" — collapsed by default on the page I read.

Internal evals built by the team shipping the model are the weakest form of evidence. So I wanted an **external, pre-existing, human-labeled benchmark with a published leaderboard that Jev's makers had no hand in constructing.**

Grounded fact-checking is close to an ideal test case for Jev, and I want to be upfront that this is a *favorable* choice, not a neutral one:

1. **It's natively a `noul` question.** "Is this claim supported by this document?" is a binary proposition. No prompt engineering gymnastics required to force Jev's shape onto the task.
2. **It's the canonical guardrail workload.** If Jev's use case is "verify everything — score, judge, verify, guardrail," then hallucination detection on grounding documents *is* the use case, not a proxy for it.
3. **Volume is the point.** As the LLM-AggreFact authors put it: "A response from an LLM might consist of many sentences. To identify and localize errors, a fact-checker needs to be called many times. If we use GPT-4 as the fact-checker, we can easily spend >10x more to verify the response than we did to produce it in the first place!" A 100x cost reduction changes what you can afford to check.
4. **There's a real leaderboard with 39 models on it**, including frontier LLMs and specialized models down to 0.4B parameters, so "good" has an unambiguous meaning.

### The MiniCheck paper

LLM-AggreFact was introduced in **"MiniCheck: Efficient Fact-Checking of LLMs on Grounding Documents"** (Liyan Tang, Philippe Laban, Greg Durrett — EMNLP 2024, pages 8818–8847). It's worth reading on its own terms, because the paper's thesis is almost exactly TypeSafe's, arrived at two years earlier by a different route:

> "we show how to build small fact-checking models that have GPT-4-level performance but for 400x lower cost."

Their method is synthetic data generation with GPT-4, in two directions:

- **C2D ("claim to doc")** — take a human-written claim, decompose it into atomic facts with GPT-3.5, then have GPT-4 expand each atomic fact into a *pair* of sentences designed such that the fact is supported **if and only if information from both sentences is combined**. Then generate a document containing those sentence pairs. Delete one sentence of a pair to produce a negative. This forces multi-sentence reasoning rather than lexical matching.
- **D2C ("doc to claim")** — take real documents (~300 Google News articles), chunk them into thirds, summarize each chunk with GPT-4, then perturb: delete sentences from the chunk and re-label, and cross-pair claims against *other* chunks of the same document.

14K synthetic examples (7,076 C2D + 7,319 D2C) plus a 21K ANLI subset, and **MiniCheck-FT5 (Flan-T5-Large, 770M params)** hits 74.7 average BAcc against GPT-4's 75.3 — at $0.24 versus $107 to decode the test set.

The benchmark itself is a deliberate aggregation. The inclusion criteria are strict, and I think this is the most valuable thing about it:

> "all datasets contain human-annotated (document, claim, label) tuples. The documents come from diverse sources, including Wikipedia paragraphs, interviews, web text, covering domains such as news, dialogue, science, and healthcare. The claims to be verified are mostly generated from recent generative models (except for one dataset of human-written claims), **without any human intervention in any format, such as injecting certain error types into model-generated claims.**"

That last clause is why they *excluded* HaluEval and SummEdits — those datasets prompt a model to intentionally make errors, and "these errors are unnatural and do not fit with our goal of detecting true LLM generation errors." They also dropped FActScore because a non-negligible fraction of its labels appear wrong from the standpoint of grounded fact-checking specifically.

The paper covers **10** datasets (~12,949 test examples, which they round to "13K"). The live leaderboard added **RAGTruth** afterwards, bringing it to **11 datasets and 29,320 test examples** — RAGTruth alone is 16,371 of them, more than half the benchmark. My run uses the 11-dataset version, which is what the leaderboard scores against.

The four source categories:

| Type | Datasets |
|---|---|
| Summarization | AggreFact-CNN/XSum, TofuEval-MediaS/MeetB, RAGTruth |
| Retrieval-augmented generation | ClaimVerify, LFQA, ExpertQA, RAGTruth |
| Post-hoc grounding | ExpertQA, REVEAL, FactCheck-GPT |
| Human-written claims | WiCE |

### The metric

Balanced accuracy: `BAcc = ½ (TP/(TP+FN) + TN/(TN+FP))`.

Small honest note: the paper justifies this purely by citation to prior work — two sentences, no argument. The obvious motivation is label imbalance, and the data supports it (the negative rate across datasets ranges from 10% on AggreFact-CNN to 77% on REVEAL), but the authors don't actually make that argument in print, so I won't attribute it to them.

The leaderboard "Average" column is the **unweighted macro average across the 11 datasets** — I verified this by recomputing Bespoke-Minicheck-7B's row (851.5 / 11 = 77.4 ✓). This matters a lot: it means AggreFact-CNN's 558 examples count exactly as much as RAGTruth's 16,371.

---

## The harness

I wrote a ~430-line Python harness that streams the HuggingFace parquet, issues one Decisions API call per example, and writes a per-call CSV (latency, input tokens, output tokens, API-reported cost, prediction, gold label) so everything is auditable after the fact.

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

This is deliberately modeled on the paper's zero-shot LLM prompt (Table 23):

> "Determine whether the provided claim is consistent with the corresponding document. Consistency in this context implies that all information presented in the claim is substantiated by the document. If not, it should be considered inconsistent."

Same protocol as the leaderboard:

- **One call per claim.** No decomposition. The paper tested atomic-fact decomposition and found "no clear indication that decomposing claims into atomic facts can consistently improve models' performance" while multiplying cost by 2–4x, so they recommend against it. I implemented a sentence-fanout mode and didn't use it for the headline number.
- **Threshold fixed at 0.5**, the midpoint of the output range, exactly as the paper does. No per-dataset tuning. The leaderboard is explicit that per-dataset threshold tuning is possible and would help, and equally explicit that they don't do it, "in order to focus on building systems that can be deployed zero-shot across multiple downstream tasks."
- **Full test set.** Not a sample. All 11 datasets, all 29,320 examples.
- **Macro average across datasets**, matching the leaderboard's Average column.

Two things I did that the leaderboard protocol does not specify: Jev doesn't take a prompt, so the `instructions`/`criteria` wording is mine. And `criteria` on a `noul` is optional; I supplied both sides, following the both-sides rule in OpenRouter's guide (describe both `true` and `false` so near-misses fall on the correct side).

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

**+1.3 over SOTA.** A 2,000-resample bootstrap over the per-example predictions puts the 95% CI at **[77.7, 79.8]**, with P(macro > 77.4) = 0.994. So the gap is unlikely to be pure sampling noise — though see the caveats below, because sampling noise is not the thing I'd worry about here.

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

Against the *per-dataset best* across all leaderboard models — a much harder bar, since no single model wins everywhere — Jev takes the top spot on 4 of 11 (WiCE, REVEAL tied, LFQA, RAGTruth) and is never more than 2.4 points off the best.

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

**Jev leans "supported."** TPR exceeds TNR on 6 of 11 datasets, dramatically so on AggreFact-CNN (95.8 vs 40.4) and TofuEval-MediaS (92.8 vs 58.1). On AggreFact-CNN it catches 96% of supported claims and misses 60% of unsupported ones. If your job is *catching hallucinations*, that specific failure mode is the one that hurts, and a 0.5 threshold is clearly wrong for it. The fix is trivial — Jev returns a probability, so you move the threshold — but it means the headline number understates what a tuned deployment would do *and* overstates what a naive one gives you.

**ExpertQA is hard for everybody.** Jev gets 59.6; the best model on the leaderboard gets 60.9; the *worst* gets 58.3. A 2.6-point spread across 39 models including GPT-4o and a 405B Llama. That column is close to noise, and since the leaderboard macro-averages, it drags every model down by the same ~1.8 points. Nobody has cracked it.

Note also that the pooled BAcc over all 29,320 examples is **82.9**, not 78.7. The difference is entirely the macro-averaging: the two AggreFact sets contribute 558 examples each but 1/11th of the score each, while RAGTruth's 16,371 examples also get 1/11th. I report 78.7 because that's the leaderboard's metric, but if you care about "what happens to a random claim from this benchmark," it's 82.9.

---

## Latency

Measured client-side, wall-clock, one request at a time, from my laptop:

| | |
|---|---|
| Median (p50) | **355 ms** |
| Mean | 611 ms |
| Mean excluding >10 s outliers | 547 ms |
| p95 | 1.89 s |
| p99 | 3.62 s |
| Min | 193 ms |
| Max | **1,031 s** |
| Total API time | 298.6 min (~5 h) |

The median sits inside TypeSafe's claimed 70–500 ms band. The tail does not. And the claim is worth reading carefully — their published evals are "generally run from our laptops on the West Coast (this is where our service is currently based)." I am not on the West Coast, so some of the 355 ms is my own round trip.

The tail is thin but real:

| Threshold | Calls over it | % |
|---|---|---|
| >5 s | 144 | 0.49% |
| >10 s | 25 | 0.085% |
| >30 s | 9 | 0.031% |
| >60 s | 5 | 0.017% |
| >120 s | 4 | 0.014% |

And one call took **17 minutes**. The slowest ten were 22s, 30s, 30s, 34s, 37s, 62s, 125s, 127s, 192s, 1031s. That single outlier alone inflates the standard deviation from 0.69 s to 6.27 s. With early-access infrastructure I'd expect this to improve, but if you're putting Jev on a latency-critical path — which is precisely the use case TypeSafe advertises — **p99.99 is what will page you, not p50.** Budget a timeout and a fallback.

Per-dataset latency also tracks document length and, apparently, load: REVEAL (489 avg input tokens) averaged 1.00 s while LFQA (787 tokens) averaged 0.34 s, so input size is not the whole story.

### The throughput caveat that matters

My run took ~5 hours of serial API time: **1.64 examples/second**.

Bespoke-MiniCheck-7B does the same 29,320 examples in **~50 minutes on a single NVIDIA A6000** — and ~30 minutes with automatic prefix caching enabled, since many LLM-AggreFact datasets reuse the same document across claims. That's **>500 docs/minute, or ~8.3/second**, five times my serial rate.

So: **Jev wins decisively on per-call latency and loses on batch throughput, at least the way I ran it.** Those are different properties for different jobs. If you need a verdict inside a request/response cycle, 355 ms from an API with no GPU to provision is a very different offer than a batched local model. If you need to grind through a million claims offline, a 770M model on one GPU with prefix caching is going to eat Jev's lunch. My harness was strictly sequential; concurrent requests would close most of this gap, and I didn't test it.

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

Read that table carefully, because it cuts both ways.

**Against LLMs, the claim holds.** 64–188x cheaper than using a frontier chat model as a fact-checker, and my `noul` costs 20 output tokens where a reasoning model would burn hundreds. The paper's argument — that verifying a response with GPT-4 can cost 10x more than generating it — mostly evaporates at $0.044 per 1,000 claims. You can afford to check every sentence of every response.

**Against a self-hosted small model, it does not.** MiniCheck-Flan-T5-L is *2.4x cheaper per example* than Jev if you already have a GPU, at 75.0 versus 78.7 BAcc. The paper's Table 3 puts it at $0.24 for 12,949 examples on a $0.80/hr A6000, which scales to ~$0.54 for the full set. FactCG-DeBERTa-L is 0.4B parameters and scores 75.6.

So the honest framing of Jev's cost story on this task is **not** "cheapest." It's:

> +3.7 BAcc over the best sub-1B open model, +1.3 over the 7B SOTA, at 2.4x the marginal cost of the former, with zero infrastructure, no model to download, no GPU to rent, no vLLM to configure, and 355 ms from a cold `curl`.

For most teams that's a better trade than $0.54. For anyone running fact-checks at real volume with a GPU already in the rack, it isn't. Notably, the LLM-AggreFact authors anticipated exactly this axis: "We think it's important for LLM fact-checkers to be small and cheap to run," and they put model size in the leaderboard as a first-class column. Jev's size is undisclosed, which means it can't be placed on that axis at all.

---

## Limitations

This is the part I care about most, and it's where the enthusiasm should get spent.

### 1. I cannot see why Jev decided anything

This is the big one, and it is *architectural*, not a missing feature. Jev returns a number. There is no reasoning trace, no cited span, no explanation, because the model does not generate text. When Jev says a claim is 0.31 supported, I have no idea whether it found a genuine contradiction, got confused by a date, or pattern-matched on something irrelevant.

Every error in my 29,320 rows is opaque. I can tell you Jev has 40.4% TNR on AggreFact-CNN. I cannot tell you *why*, and no amount of staring at the output will tell me. With an LLM I'd read 30 chain-of-thoughts and have a hypothesis in an hour.

Interestingly, the MiniCheck authors list this as a limitation of *their* models too:

> "Like many other specialized fact-checking models, our models do not reveal their internal decision-making processes, making it challenging to localize errors to particular mismatched spans of a claim or document."

Their suggested mitigation is decomposition: fan out per atomic fact, and at least you learn *which* fact failed. That works for Jev too — ask N `noul` questions in one parallel request, one per sentence or per atomic fact, and the answer vector localizes the failure at no latency cost and near-zero marginal token cost. I built this mode (`--mode fanout`) and didn't use it for the headline number, because the paper shows decomposition doesn't reliably improve accuracy and I wanted protocol parity. But for a production guardrail I'd almost certainly run it: interpretability is essentially free, since Jev evaluates all questions in parallel.

Still — localizing *which* fact failed is not the same as knowing *why*. If you need to show a human reviewer a reason, or defend a decision, or debug a systematic error, Jev alone cannot do it. You need an LLM in the loop for that, which puts you back at LLM latency and LLM cost for the subset of cases where you need an explanation.

### 2. I did not log the probabilities

My own methodological error, and it's a bad one. The harness thresholded at 0.5 and wrote `pred` to CSV, throwing away the raw `noul` value. Which means from 29,320 API calls I cannot produce:

- a calibration curve, which is the *single most interesting property* Jev claims ("higher confidence means higher accuracy")
- a threshold sweep or per-dataset tuning
- an ROC curve or AUC, which would be a threshold-free comparison
- any analysis of whether the AggreFact-CNN TNR collapse is a threshold artifact or a real capability gap

RLCD is supposedly the whole innovation, and calibration is the whole point of RLCD, and I measured accuracy at one arbitrary threshold. **If you replicate this, log the probability.** I'll re-run with it; at $1.29 a pass there is genuinely no excuse.

### 3. Contamination is unruled-out and unrulable-out

LLM-AggreFact ships a `contamination_identifier` field precisely because this is a known hazard. The dataset has been public on HuggingFace since 2024. Jev 1.13 was trained in 2026. TypeSafe does not disclose their training data.

I have no way to check. A 78.7 on a public benchmark from a model whose training set I can't inspect is weaker evidence than it looks, and this applies to every frontier model on that leaderboard, not just Jev. It's the reason TypeSafe's own workflow evals exist. I'd note the direction of the result is at least mildly reassuring — Jev is weakest on ExpertQA and AggreFact-XSum, and a contaminated model would presumably be uniformly strong — but that's a vibe, not a test.

### 4. The prompt is mine

The leaderboard protocol specifies a prompt (Table 23). Jev doesn't accept prompts; it accepts `instructions` and `criteria`. I wrote one version, modeled on the paper's, and ran it 29,320 times. I did not sweep alternatives.

This is a real asterisk on "apples-to-apples." It's possible a better-worded `noul` gets 80. It's equally possible I got lucky and a neutral phrasing gets 76. TypeSafe's own guidance is that criteria wording does a lot of work — "Jev applies both literally," describe both sides, use the vocabulary present in the input text — which implies the wording is a meaningful free parameter that the LLM entries on the leaderboard also enjoy but that nobody audits.

### 5. Single run, single machine, one model version

n=1. No repeat runs, so I can't separate real variance from network variance. Latency was measured from one laptop on one network on one evening, against early-access infrastructure that is presumably under active load from everyone else who just got off the waitlist. The model string in responses is a dated build (`typesafe/jev-1.13-20260917`), so these numbers have a shelf life.

### 6. This is one task shape, and a flattering one

Grounded fact-checking is a *single binary proposition over a supplied document*. That is the best possible case for a model whose output type is a single binary proposition over a supplied state. It says nothing about `choice` with 200 options, `score` calibration, multi-question workflows, or anything requiring arithmetic, exact counting, or date comparison — all of which TypeSafe's own [jaggedness page](https://docs.typesafe.ai/model-jaggedness/jev-1.13) explicitly tells you to keep in code.

Do not read "#1 on LLM-AggreFact" as "#1 at classification." Read it as "#1 at this one benchmark, which happens to be a really important one if you're building RAG guardrails."

### 7. The macro average is a choice

The leaderboard macro-averages 11 datasets of wildly different sizes. Jev's pooled BAcc is 82.9; its macro is 78.7. Its biggest wins are on RAGTruth (+2.9, 16,371 examples) and TofuEval-MeetB (+5.5, 772 examples), which count equally. I used the leaderboard's metric because that's the only way to compare, but the ranking is somewhat metric-dependent and I can't recompute pooled numbers for the other models.

---

## What I actually think

The headline is real: **Jev 1.13 scores 78.7 macro BAcc on the full LLM-AggreFact test set, which is the best published number on that leaderboard, for $1.29 and a median of 355 ms per call, with no GPU and no fine-tuning.** I did not tune a threshold, sample a subset, or sweep prompts to get there. Anyone with an OpenRouter key can check it for the price of a coffee.

But the interesting finding isn't the rank. It's the shape of the trade:

- Against **frontier LLMs as fact-checkers**, Jev is a straightforward win — better accuracy, 64–188x cheaper, and a 355 ms median against the 3–329 s end-to-end range TypeSafe cites for frontier models. The MiniCheck paper's core complaint, that verification costs more than generation, stops being true.
- Against **purpose-built small open models**, it's genuinely close. +3.7 BAcc over MiniCheck-Flan-T5-L at 2.4x the marginal cost and 1/5th the batch throughput, in exchange for zero infrastructure. That's a real decision with a real answer that depends on your volume and whether you own GPUs.
- The **interpretability cost is not a footnote.** It's the defining property. You are buying a number, and if you need a reason you need something else. The mitigation — parallel decomposed `noul`s that localize the failure for free — is good, and it is not the same as an explanation.
- The **tail latency** is the thing that would stop me shipping this on a critical path today without a fallback. p50 of 355 ms is great. A 17-minute call is not.

The most compelling thing about Jev isn't that it beat a 7B model at fact-checking. It's that at $0.044 per 1,000 claims, checking *every sentence of every LLM response in production* goes from a line item to a rounding error. That's a change in what's possible, not a change in a benchmark number.

I'm going to re-run this with probabilities logged, a per-dataset threshold sweep, and a concurrency setting, and post the calibration curves. That's the experiment that would actually test the claim RLCD is making.

---

## Reproduce it

```bash
# ~29,320 calls, ~$1.29, ~5h serial
python jev_harness.py --mode single

# project cost before spending anything
python jev_harness.py --estimate
```

The harness streams `lytang/LLM-AggreFact` test split from HuggingFace, writes a per-call CSV (`dataset, example_idx, mode, latency_s, input_tokens, output_tokens, cost_usd, pred, gold`), supports `--resume` from a partial CSV, and retries 429/5xx with exponential backoff. You need an `OPENROUTER_API_KEY` and an `HF_TOKEN`.

*(Add the probability column. Learn from my mistake.)*

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