---
permalink: /
title: "M. Mikail Demir"
author_profile: true
redirect_from:
  - /about/
  - /about.html
---

I am a researcher working on natural language processing and AI for the legal domain, with a focus on legal argument mining, fact-checking, and the privacy-preserving deployment of large language models. I am based at the University at Albany, SUNY.

## Research Interests

My work sits at the intersection of NLP, AI, and law, and most of it comes back to one question: where do LLMs fail, and why does that matter in high-stakes settings?

- **LLM failure modes** — Hallucination is the headline failure, but it's not the only one. I'm interested in the broader space of ways large language models get things wrong with confidence: calibration failures, threshold sensitivity, and the gap between what a model scores on a benchmark and what it does in deployment.
- **Hallucination** — Grounded fact-checking is the specific lens I work through. Detecting when a claim is not supported by its source document is a well-defined task with a real leaderboard, and it's the canonical guardrail workload for RAG systems.
- **Legal NLP** — Legal text is long, structural, and argumentative in ways that general-domain NLP doesn't handle well. I work on argument mining, precedent treatment classification, and extractive QA over regulatory and competition decisions.
- **Applied AI** — I care less about capability and more about deployment. Privacy-preserving LLM use in law practice, cost-efficient guardrails that can run on every sentence, and the engineering trade-offs between frontier LLMs, small specialized models, and new architectures like System One Models.
- **Impact of AI on the legal profession** — The adoption question. Lawyers are risk-averse, confidentiality is non-negotiable, and getting something subtly wrong has consequences. I'm interested in what actually changes practice versus what just benchmarks well.
- **LLM benchmarking** — Building and running external, human-labeled benchmarks that a model's makers had no hand in constructing. Internal evals are the weakest form of evidence; the interesting work is designing evaluations that are hard to game and that say something about real deployment.
