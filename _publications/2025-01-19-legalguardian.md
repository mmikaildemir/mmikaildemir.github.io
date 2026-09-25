---
title: "LegalGuardian: A Privacy-Preserving Framework for Secure Integration of Large Language Models in Legal Practice"
collection: publications
category: manuscripts
permalink: /publication/2025-01-19-legalguardian
excerpt: 'Large Language Models (LLMs) hold promise for advancing legal practice by automating complex tasks and improving access to justice. However, their adoption is limited by concerns over client confidentiality. LegalGuardian employs Named Entity Recognition (NER) techniques and local LLMs to mask and unmask confidential PII within prompts, achieving a F1-score of 93% with GLiNER and 97% with Qwen2.5-14B in PII detection.'
date: 2025-01-19
venue: 'arXiv preprint arXiv:2501.10915'
paperurl: 'https://arxiv.org/abs/2501.10915'
citation: 'M. Mikail Demir, Hakan T. Otal, M. Abdullah Canbaz. (2025). &quot;LegalGuardian: A Privacy-Preserving Framework for Secure Integration of Large Language Models in Legal Practice.&quot; <i>arXiv preprint arXiv:2501.10915</i>.'
---

Large Language Models (LLMs) hold promise for advancing legal practice by automating complex tasks and improving access to justice. However, their adoption is limited by concerns over client confidentiality, especially when lawyers include sensitive Personally Identifiable Information (PII) in prompts, risking unauthorized data exposure. To mitigate this, we introduce LegalGuardian, a lightweight, privacy-preserving framework tailored for lawyers using LLM-based tools. LegalGuardian employs Named Entity Recognition (NER) techniques and local LLMs to mask and unmask confidential PII within prompts, safeguarding sensitive data before any external interaction. We detail its development and assess its effectiveness using a synthetic prompt library in immigration law scenarios. Comparing traditional NER models with one-shot prompted local LLM, we find that LegalGuardian achieves a F1-score of 93% with GLiNER and 97% with Qwen2.5-14B in PII detection. Semantic similarity analysis confirms that the framework maintains high fidelity in outputs, ensuring robust utility of LLM-based tools.

Recommended citation: M. Mikail Demir, Hakan T. Otal, M. Abdullah Canbaz. (2025). "LegalGuardian: A Privacy-Preserving Framework for Secure Integration of Large Language Models in Legal Practice." <i>arXiv preprint arXiv:2501.10915</i>.