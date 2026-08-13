# Response retrieval & re-ranking for conversational AI

**Role:** Master's Thesis, Boğaziçi University (2019–2022)
**Finding:** A smaller, task-specific re-ranking model outperformed larger general-purpose alternatives on conversational response retrieval, and the approach extended to open-domain QA without task-specific fine-tuning.
**Related publication:** *Intent Classification with Short Sentences in Agglutinative Languages* (2020) — adjacent NLP work from the same research period, on intent detection in morphologically rich languages such as Turkish.

---

## Problem

Given a conversational context, retrieve the best next response from a large candidate pool. This is the retrieval half of retrieval-based dialogue systems: rather than generating a response token by token, the system selects the most appropriate one from a fixed or large candidate set. The research question was specifically about the *retrieval and ranking* step — how to find the right response accurately and efficiently when the candidate pool is large enough that scoring every candidate with a heavyweight model is impractical.

## Constraints

- **Candidate pools too large for exhaustive neural scoring.** Running a full cross-encoder — a model that jointly encodes the context and each candidate together — over every candidate for every query doesn't scale; cross-encoders are accurate but expensive per comparison.
- **Needed to work without task-specific fine-tuning for the QA extension.** Part of the research question was whether a system built for conversational retrieval could generalize to open-domain question answering without retraining for that new task.
- **Academic compute budget**, not industrial-scale infrastructure — the architecture had to be efficient by design, not efficient because of a large compute budget.

## Architecture

A two-stage pipeline, which is the standard pattern for making neural re-ranking tractable at scale:

1. **Stage 1 — TF-IDF candidate retrieval.** A fast, classical sparse-retrieval method narrows a large candidate pool down to a much smaller shortlist. TF-IDF is cheap to run over the full pool and doesn't require a forward pass through a neural network, which is what makes the pipeline tractable at scale.
2. **Stage 2 — Cross-Encoder re-ranking.** The shortlist from Stage 1 is re-scored by a Cross-Encoder, which jointly attends over the context and each candidate response — far more accurate than scoring context and candidate independently (as a bi-encoder / embedding-similarity approach would), but only affordable because Stage 1 already cut the candidate count down.

This retrieve-then-rerank pattern is the same shape used in modern production RAG systems — cheap broad retrieval followed by an expensive precise re-ranker over a shortlist — which is part of why the thesis work has stayed directly relevant to production system design since.

## Key decisions and trade-offs

- **Two stages instead of one.** A single-stage Cross-Encoder over the full pool would have been more accurate per-query but computationally infeasible at pool sizes relevant to real conversational systems. A single-stage TF-IDF-only system would have been fast but left accuracy on the table. The two-stage split was the deliberate trade-off between the two failure modes.
- **Investigating smaller task-specific models rather than defaulting to larger general-purpose ones.** The central empirical finding was that a smaller, purpose-built Cross-Encoder for this retrieval task outperformed larger general-purpose transformer models on the same benchmark — the task-specific inductive bias mattered more than raw parameter count for this problem shape. That result is the part of the thesis most worth citing on its own.
- **Extending to QA without fine-tuning, as a generalization test.** Rather than treating conversational retrieval and QA as unrelated problems each requiring their own trained model, the thesis tested whether the same retrieval/re-ranking architecture transferred to open-domain QA directly — a stronger and harder claim than reporting in-domain results alone, since it tests whether the approach learned something general about relevance ranking rather than something specific to dialogue data.

## Results

The two-stage TF-IDF + Cross-Encoder pipeline retrieved and ranked conversational responses more effectively than baseline single-stage approaches, and the smaller task-specific Cross-Encoder outperformed larger general-purpose transformer baselines on the retrieval/re-ranking task. The architecture extended to open-domain QA and produced strong results there without task-specific fine-tuning, supporting the conclusion that the retrieval/re-ranking pattern generalizes beyond the dialogue setting it was designed for.

## What I'd do differently

- **I'd push harder on error analysis by candidate difficulty**, rather than aggregate accuracy alone — understanding *which* candidates the Cross-Encoder still got wrong after re-ranking would have pointed more directly at where the model's inductive biases were failing.
- **I'd test more than one classical Stage-1 retriever.** TF-IDF was a reasonable, well-understood choice, but comparing it against BM25 or a lightweight dense retriever as the first stage would have clarified how much of the final performance was coming from Stage 1's recall versus Stage 2's precision.
- **I'd design the QA generalization test with a harder distribution shift.** The QA extension showed the architecture wasn't purely memorizing dialogue-specific patterns, but a more deliberately out-of-domain QA benchmark would have made that generalization claim more conclusive.
