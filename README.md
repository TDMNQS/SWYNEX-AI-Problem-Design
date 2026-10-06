# SWYNEX Task 1 — AI Problem Design

**Project:** Evidence-Based Question Answering over Five Research Papers  
**Prepared by:** Numan Qureshi | B.Tech Information Technology, MGM University  
**Date:** 6 October 2026  
**Status:** Problem design completed; implementation and evaluation are proposed.

## 1. Problem and target user

Undergraduate students beginning a literature review need to locate specific methods, assumptions, and limitations in research papers. Reading every paper for each question takes time, and an answer without traceable evidence is difficult to verify.

Design a small assistant that answers English questions using only a fixed collection of five papers about retrieval and language models. Each answer must identify its supporting paper and passage. When the collection does not contain sufficient evidence, the assistant should explicitly decline to answer.

**User story:** As a final-year IT student, I want to ask a focused question about a paper and inspect the supporting passage so that I can verify the answer before using it in my literature review.

**Example input:** “How does Sentence-BERT make semantic similarity search efficient?”  
**Expected behavior:** A concise explanation based on the relevant passage, with paper title, section, PDF page, and evidence snippet. This is a desired output, not a recorded model result.

## 2. Scope and data source

The proposed corpus contains the main body of these five public research papers. The original papers remain authoritative; their findings are not claims about this proposed assistant.

| ID | Paper | Source |
|---|---|---|
| P01 | Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks | https://arxiv.org/abs/1908.10084 |
| P02 | Dense Passage Retrieval for Open-Domain Question Answering | https://arxiv.org/abs/2004.04906 |
| P03 | Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks | https://arxiv.org/abs/2005.11401 |
| P04 | REALM: Retrieval-Augmented Language Model Pre-Training | https://arxiv.org/abs/2002.08909 |
| P05 | Lost in the Middle: How Language Models Use Long Contexts | https://arxiv.org/abs/2307.03172 |

This document selects and verifies source metadata; the corpus has not yet been downloaded or indexed. Before implementation, record the exact arXiv version, retrieval date, SHA-256 checksum, and license for each PDF. Use permitted local copies; publish the manifest and links rather than redistributing PDFs without checking their licenses.

**Included:** English questions about prose descriptions of methods, motivations, and limitations within the five papers; short comparisons supported by passages from both papers.

**Excluded:** Web search, new-paper discovery, full citation-network extraction, scanned-PDF OCR, image interpretation, equation derivations, and questions that require interpreting tables. Claims appearing only in references or cited external papers are outside the corpus.

### Data preparation plan

1. Extract selectable PDF text while preserving the original PDF page index.
2. Remove repeated headers and footers; normalize whitespace and hyphenation. Check sample pages against the PDFs, particularly multi-column reading order.
3. Exclude the reference list from retrieval. Keep section boundaries and label unusable equation/table text.
4. Split prose into approximately 350-token chunks with 50-token overlap, without crossing paper boundaries. Merge very short fragments with adjacent prose.
5. Store `paper_id`, `source_url`, `version`, `chunk_id`, `section`, `pdf_page_start`, `pdf_page_end`, and `text`. Page citations refer to PDF page indices, not printed journal page numbers.

## 3. Proposed AI approach and baseline

The proposed method is retrieval-augmented generation (RAG). Retrieval narrows the evidence available to a generator; generation turns selected passages into a readable answer. AI is useful for paraphrased questions and synthesis, while a lexical baseline tests whether the added complexity is justified.

**Baseline:** BM25 over identical chunks, returning the top five passages without a generated answer. Manual PDF search is the usability reference.

**Candidate:** Encode questions and chunks using one fixed pretrained sentence-embedding model. Rank by cosine similarity; use an in-memory vector matrix for this small corpus. Return at most five chunks to one fixed language model. Record model identifiers, revisions, prompts, retrieval settings, and generation settings. No fine-tuning is planned.

**Generation rules:** Use only the supplied evidence; attach chunk identifiers to factual claims; preserve reported qualifications; treat document text as data rather than instructions. If evidence is insufficient or ambiguous, say “I cannot answer this from the selected papers.” Citations must resolve to supplied chunks. A citation being present does not prove support: reviewers must check the cited passage.

**Answer schema:** `answer`, `abstained`, `citations[]` containing paper ID, title, section, PDF pages, chunk ID, and a short supporting snippet. Retrieval scores are ranking signals, not calibrated probabilities.

**Workflow:** Question → retrieve passages → check evidence sufficiency → generate an evidence-bound answer or abstain → validate citation IDs → display answer and source passages. Any retrieval-score threshold must be selected on development questions, then frozen before testing.

## 4. Constraints and failure handling

| Constraint or risk | Design response |
|---|---|
| Only five papers; no external knowledge | Answer exclusively from indexed prose and disclose corpus scope. |
| Hallucinated or misattributed claims | Require claim-level citations and manually score support. |
| Multi-column extraction errors | Inspect sampled pages; repair or exclude malformed chunks before evaluation. |
| Similar terminology across papers | Show paper titles and sections alongside citations. |
| Unsupported or ambiguous questions | Abstain or ask for clarification rather than infer missing facts. |
| Prompt-like instructions inside PDFs | Treat retrieved content as evidence, never as executable instructions. |
| CPU-only student laptop | Precompute embeddings; use an in-memory index; no GPU requirement. |
| API cost and latency | Limit evidence to five chunks, answers to 250 words, and one generation call per question; log token use and actual cost. |
| Model/API changes | Pin versions where possible and record execution dates and provider settings. |
| Privacy | Use public papers; avoid personal data in questions and logs; keep credentials out of the repository. |

Planning limits: total prototype API spend at most US$5, and at most 10 seconds p95 end-to-end warm-query latency on a documented machine and network. These are design budgets, not verified pricing or measured performance. If the budget is unavailable, evaluation can first cover retrieval alone; the complete answer-generation criteria would remain untested.

## 5. Evaluation plan

Create **50 human-authored questions after inspecting the corpus**: 35 answerable and 15 intentionally unanswerable. Include definitions, methods, limitations, paraphrases, and comparisons. Give each answerable question a concise reference answer and all identified relevant evidence chunks. Give each unanswerable question a documented reason why the fixed corpus cannot answer it.

Use 20 development questions (14 answerable, 6 unanswerable) to tune chunking, prompts, and thresholds. Freeze the configuration, then run the remaining 30 held-out questions (21 answerable, 9 unanswerable). Keep paraphrases and closely related questions in the same split. Papers may be present in both splits because the test measures new questions over a fixed corpus, not generalization to new papers. Never use test answers to tune the system.

### Metrics and proposed success criteria

| Metric | Definition and held-out target |
|---|---|
| Evidence Hit@5 | Fraction of 21 answerable questions with at least one gold evidence chunk in the top five; ≥18/21. |
| Recall@5 | Per-question fraction of all gold evidence chunks retrieved in the top five, averaged over answerable questions; report with Hit@5, especially for comparisons. |
| Answer correctness | Human rubric: 0 incorrect/unsupported, 1 partly correct or incomplete, 2 correct and sufficiently complete; ≥17/21 score 2. Abstentions on answerable questions score 0. |
| Citation support | Supported factual claims / all factual claims in emitted answers; ≥95%. Missing citations count as unsupported. Report numerator and denominator. |
| Correct abstention | At least 8/9 unanswerable questions correctly declined; a fabricated answer fails. |
| Answerable coverage | At least 19/21 answerable questions receive a substantive answer rather than an abstention. |
| Latency | p95 warm-query time ≤10 seconds across all 30 held-out queries; report cold-start time separately. |
| Budget | Actual cumulative prototype API spend ≤US$5, with token and cost logs. |

For multi-paper answers, Hit@5 alone is insufficient: also report whether evidence from every required paper was retrieved. A second reviewer should independently score at least 10 test questions; resolve disagreements against the original passages. Report counts, denominators, and disagreement rate. These small-sample results would be exploratory, not proof of broad reliability.

Compare BM25 and dense retrieval on the same questions and chunks. Prefer the simpler baseline if dense retrieval provides no useful improvement. Diagnose failures separately as extraction, retrieval, evidence selection, generation, citation, or abstention errors.

## 6. Illustrative evaluation cases

These are example question specifications, not a completed benchmark or measured outputs. Gold answers and exact evidence locations must be annotated from the selected PDF versions.

| Case | Question | Expected behavior |
|---|---|---|
| E01 | What similarity measure does Sentence-BERT use to compare its sentence embeddings? | Answer from P01 and cite the relevant passage. |
| E02 | What role does the retriever play in the RAG approach? | Answer from P03 with supporting evidence. |
| E03 | How do DPR and Sentence-BERT describe their different retrieval or embedding objectives? | Compare only supported prose from P01 and P02; cite both. |
| E04 | What problem does Lost in the Middle investigate about long contexts? | Answer from P05 and preserve the study's stated scope. |
| E05 | What is the latest 2026 benchmark score for this assistant? | Abstain: no assistant results or current benchmark data exist in the selected corpus. |
| E06 | What private training examples were used by an unnamed commercial model? | Abstain: the question has no supporting evidence in this corpus. |

## 7. Deliverables and acceptance

Task 1 deliverable: this problem-design document, also provided as a PDF. It defines the user, data sources, boundaries, proposed solution, constraints, and measurable evaluation criteria. A working application is a future implementation phase.

Future implementation artifacts: versioned corpus manifest; extraction/chunking pipeline; BM25 baseline; dense retriever; evidence-bound answer generator; annotated development/test questions; evaluation report and error analysis.

**Current results:** No prototype has been executed, no benchmark has been scored, and no success target has been demonstrated. Completion here means the requested problem design is documented.

## 8. Submission steps

1. Create a public repository named `SWYNEX-AI-Problem-Design` in your GitHub account.
2. Upload `README.md` and `SWYNEX_AI_Problem_Design.pdf`. The README is the accessible design document.
3. Open the repository URL while signed out and confirm the document can be read.
4. Publish the supplied LinkedIn draft. Select the official SWYNEX Technologies page when tagging it, and attach a screenshot of the document's first page or a short explanation video.
5. Paste the actual repository URL and published LinkedIn post URL into the two task fields, then submit.

Do not submit a local download link as the public document URL. Publishing and portal submission are not performed by this document.
