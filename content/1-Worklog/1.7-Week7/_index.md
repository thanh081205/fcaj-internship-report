---
title: "Week 7 Worklog"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.7. </b> "
---
### Week 7 Objectives:

* Build the complete `search_service` ML pipeline locally: transcription, captioning fallback, embeddings, vector storage, and hybrid search.
* Benchmark different retrieval strategies and pick the best one for production.
* Fix data-quality and API-compatibility bugs found during testing.

### Tasks carried out this week:
| Day | Task | Date | Reference Material |
| --- | --- | --- | --- |
| 1 (Mon) | Implement **transcription** with Whisper (`faster-whisper`, "small" model), with `no_speech_prob` filtering at both video and chunk level | 13/07/2026 | |
| 2 (Tue) | Implement **BLIP** image-captioning fallback for videos with no audio track or silent audio | 14/07/2026 | |
| 3 (Wed) | Implement **embeddings**: `multilingual-e5-large` (1024-dim dense vectors) + BM25/IDF sparse model; store in **Qdrant** (`video_chunks` collection, `uuid5` point IDs) with `userId`/`visibility` propagated end-to-end through RabbitMQ → Qdrant (owner always sees their content; others only see PUBLIC) | 15/07/2026 | |
| 4 (Thu) | Implement and benchmark 3 **search methods** — dense-only, sparse-only (BM25), and hybrid RRF fusion — using Precision@3, Recall@3, and MRR | 16/07/2026 | |
| 5 (Fri) | Debug data-quality issues found during benchmarking; write up findings in a benchmark report (`bao-cao-dense-sparse-hybrid.md`) for the team | 17/07/2026 | |

### Week 7 Achievements:

* Completed a fully working `search_service` ML pipeline locally, end to end (ingest → transcribe/caption → embed → index → search).
* Found that **dense-only retrieval outperforms hybrid** (MRR ≈ 0.900 vs ≈ 0.684): unweighted RRF fusion degrades when the sparse (BM25) side fails on cross-lingual queries, so dense-only was chosen as the safer default for this use case.
* Fixed a **diacritics-symmetry bug** in BM25: stripping Vietnamese diacritics at query time while indexing raw accented text broke retrieval; fixed by applying `normalize_transcript_text()` (diacritics-preserving, `re.UNICODE`) symmetrically at both indexing and query time.
* Found and tracked a breaking change in `qdrant-client==1.18.0` (`FusionQuery` API: `function=` → `fusion=`) affecting `search_points()`.
* Fixed a variable-shadowing bug caused by naming both the e5 and Whisper models `model`; renamed to `e5_model`/`whisper_model` for clarity.
* Learned that **cosine similarity has a ~0.82 noise floor** for `multilingual-e5` regardless of text content — an expected model-level property (anisotropy), not a bug.
* Produced a benchmark report for the team documenting the dense vs. sparse vs. hybrid comparison and the recommendation to use dense-only in production.
