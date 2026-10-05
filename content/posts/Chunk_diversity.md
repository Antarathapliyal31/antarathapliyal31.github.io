+++
title = "Chunk diversity in RAG: three mechanisms I thought through"
date = 2026-10-05
description = "Metadata filtering, parent caps, and validation agents: how I'd keep retrieved chunks from clustering in my Abilify RAG pipeline."

[taxonomies]
tags = ["rag", "retrieval", "llm", "agents"]
+++

I kept assuming hybrid search with Cohere reranker would naturally return diverse chunks. It mostly does, but not always, and the failure mode matters.

This came up in feedback on my Abilify RAG project. Someone pushed me on what happens when the top-K chunks all cluster around one section of the document. I'd thought about this casually but hadn't worked through the tradeoffs. This is my attempt to.

## The concentration problem

My Abilify setup uses parent-child chunking. Child chunks are 300 tokens with 100 token overlap. Parent chunks are 1000 tokens. With overlap, each parent holds about 4-5 children.

Hybrid search returns 10 candidates after BM25 + dense retrieval + RRF fusion. In the extreme case, those 10 could come from just 2 parents — 5 children each. That's redundant context from 2 locations in the document, not 10 useful signals.

For pharma clinical documents, this matters more than it does elsewhere. Adjacent sections carry safety-critical context. A query about dosing benefits from nearby warnings and interactions. A query about one drug interaction benefits from the general interactions table. Concentrated retrieval misses this.

Cohere reranker picks the top 3 from those 10 before sending to the LLM. If the input to rerank is already concentrated, rerank's output will be too — it's optimizing for relevance within the candidate pool, not for coverage.

## Three mechanisms I thought about

### Metadata filtering

I had this in Abilify. Each chunk carried `parent_id`, `patient_population`, `drug_name`, and `topic`. The supervisor extracted intent from the query — mentions of "elderly" triggered a `patient_population` filter, mentions of "interactions" triggered a `topic` filter. Chroma applied the filters before vector search.

What I realized: metadata filtering is primarily for precision, not diversity. It narrows the search space to relevant sections before retrieval even runs. It doesn't ensure the retrieved set spans different parts of what's left.

What I'd add if I rebuilt: explicit FDA section schema with subsections, multi-value `topic_tags` so a chunk discussing geriatric cardiac interactions belongs to all three categories, severity tags to prioritize critical warnings, document version for audit traceability.

But none of this is a diversity mechanism. It's upstream of diversity.

### Parent cap

This is the one I didn't do and probably should have.

The idea: after hybrid search returns 50 candidates, cap at 2-3 children per parent when selecting the top 10. Guarantees at least 4-5 different parents represented.

I pushed back on this initially — concentration is sometimes the right behavior. If a narrow factual query ("starting dose for adults") has its answer entirely in one parent, forcing diversity would dilute. Then I actually worked through it. For pharma clinical corpora, even narrow queries benefit from adjacent sections. The LLM doesn't need 5 near-duplicate chunks about starting dose. It needs the starting dose, the titration schedule, the elderly adjustment, and the relevant warning — spread across parents.

Hard cap at 2-3 per parent, with rerank picking 3 from the top 10, gets you distinct parents' worth of complementary context instead of one parent's content shown 5 ways.

The downstream rerank and RAGAS answer-relevancy loop catches the edge cases where cap pulls in slightly less relevant chunks. The operational cost is a small retry-rate increase, worth it when missing adjacent context is a safety issue.

### Validation agent

This is the one that came up in the feedback and the one I find most interesting.

The validation agent runs after retrieval, before generation. It's LLM-as-judge on (query, retrieved chunks). It asks: given these chunks, can the query be answered completely, with grounding and specificity? Not "is the answer good" (that's RAGAS, post-generation). "Is the retrieval sufficient" (pre-generation).

Structured output: sufficiency flag, coverage score, sub-topics missing, modified query targeting the gaps. If insufficient, re-retrieve with the modified query. Capped at 2-3 attempts. If retries exhaust, return a safe refusal — "I don't have reliable information to answer this fully" — rather than fabricating with incomplete context.

Cost: 300-600ms per query on the success path. Net savings on failures because you avoid wasted generation. If retrieval failure rate is 10%+, validation agent is roughly net-neutral in latency. Below that it's overhead.

## How they compose

Metadata narrows the pool. Parent cap diversifies the top-K. Validation confirms coverage. Each solves a different problem.

For my Abilify setup, the pipeline would be:

Metadata and rerank are precision layers. Parent cap is a diversity layer. Validation is a coverage layer. They're not alternatives, they're the three things that have to be true for retrieval to be good: it has to pull from the right sections, span those sections, and actually contain the information needed.

## What I'd test

I'd add parent cap first. It's the cheapest change — a few lines in the retrieval wrapper, no new LLM calls, measurable against the existing RAGAS gold set.

Validation agent is more interesting but costs latency I'd want to measure before committing. The right experiment: run with and without validation on a labeled query set, measure answer completeness (not just RAGAS answer relevancy — specifically whether critical sub-topics are addressed) and total latency including retries.

I'm still unsure whether validation agent and parent cap are strictly additive or whether validation makes parent cap partly redundant. Validation catches coverage gaps directly; parent cap prevents the most common cause of coverage gaps. Running one well might make the other marginal. Worth testing.

The deeper principle, maybe: diversity in RAG isn't a universal good. It's a response to concentration. The right diversity mechanism depends on which failure mode you're seeing. Pharma clinical corpora have a specific failure mode (adjacent sections carry safety-critical context), which is why parent cap is the right default there. A different corpus would call for a different mechanism, or none at all.
