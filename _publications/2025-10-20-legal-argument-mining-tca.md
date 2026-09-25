---
title: "Legal Argument Mining the TCA's Technology Undertaking Exception with Turkish BERT"
collection: publications
category: conferences
permalink: /publication/2025-10-20-legal-argument-mining-tca
excerpt: 'We model the analysis of Turkish competition law&apos;s &quot;technology undertaking&quot; exception as extractive question answering. We introduce an expert-annotated dataset of 152 Turkish Competition Authority (TCA) merger decisions in SQuAD style and fine-tune a Turkish BERT model to extract sentence-level answers. An ablation minimizing context improves the most difficult task by over 45 points, indicating context length is a primary performance bottleneck.'
date: 2025-10-20
venue: '2025 IEEE International Conference on Data Mining Workshops (ICDMW)'
paperurl: 'https://ieeexplore.ieee.org/abstract/document/11415810/'
citation: 'M. Mikail Demir, M. Abdullah Canbaz. (2025). &quot;Legal Argument Mining the TCA&apos;s Technology Undertaking Exception with Turkish BERT.&quot; <i>2025 IEEE International Conference on Data Mining Workshops (ICDMW)</i>, pp. 777-786.'
---

Understanding how enforcement authorities interpret and apply new rules requires substantial effort, as the relevant reasoning is often scattered across long and complex decisions. In Turkish competition law, understanding the practical scope of the "technology undertaking" rule (exception) requires locating specific sentences that articulate business descriptions, legal classifications, market nexus, and threshold applications. In this paper, we model this analysis as extractive question answering (QA). We introduce an expert-annotated dataset of 152 Turkish Competition Authority (TCA) merger decisions, formatted in SQuAD style with four targeted questions aligned to the TCA's reasoning. We fine-tune a Turkish BERT model to extract sentence-level answers corresponding to these four components. In 5-fold cross-validation, the model attains an overall F1 that varies by question type: the model excels on formulaic legal conclusions but struggles with implicit, substantive reasoning. An ablation that minimizes context improves the most difficult task by over 45 points, indicating that context length is a primary performance bottleneck. Taken together, the dataset and results establish a useful benchmark for Turkish legal NLP and show that model performance closely tracks the explicitness of argumentation in the source legal text.

Recommended citation: M. Mikail Demir, M. Abdullah Canbaz. (2025). "Legal Argument Mining the TCA's Technology Undertaking Exception with Turkish BERT." <i>2025 IEEE International Conference on Data Mining Workshops (ICDMW)</i>, pp. 777-786.