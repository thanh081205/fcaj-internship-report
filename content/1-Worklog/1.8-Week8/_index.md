---
title: "Week 8 Worklog"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.8. </b> "
---
### Week 8 Objectives:

* Deploy the embedding model to AWS SageMaker Serverless Inference.
* Handle a team merge conflict and cost-control setup.
* Finish testing, blog posts, and the rest of the internship report.

### Tasks carried out this week:
| Day | Task | Date | Reference Material |
| --- | --- | --- | --- |
| 1 (Mon) | Package `multilingual-e5-large` as `model.tar.gz`; write a custom `inference.py` replicating fastembed's exact behavior (query/passage prefixes, attention-mask-aware mean pooling, L2 normalization) for compatibility with already-upserted Qdrant vectors | 20/07/2026 | |
| 2 (Tue) | Set up deployment with the **SageMaker Python SDK** (`HuggingFaceModel` + `ServerlessInferenceConfig`, CPU) — the HuggingFace one-click "Deploy on SageMaker AI" button only targets real-time GPU endpoints, not Serverless Inference | 21/07/2026 | |
| 3 (Wed) | Configure **AWS Budget Alert** (~$20/month, 50%/80% actual and 100% forecasted thresholds) and enable CloudWatch billing/Free Tier alerts before going further with deployment | 22/07/2026 | |
| 4 (Thu) | Resolve a merge conflict between my `feat/semantic-search` branch and the team's `feat/refactor` branch (two independently written `search.py` files coincidentally used the same class name `SearchService`); aborted the merge, pushed my work, and deferred the architectural decision to the team | 23/07/2026 | |
| 5 (Fri) | Test the `search_service` pipeline end-to-end locally; write and publish blog posts to the AWS Study Group; finalize Self-evaluation, Sharing & Feedback, and review the whole report | 24/07/2026 – 30/07/2026 | [Blogs Posted](/3-blogsposted/), [Self-evaluation](/6-self-evaluation/), [Feedback](/7-feedback/) |

### Week 8 Achievements:

* Completed the `model.tar.gz` packaging for `multilingual-e5-large` and a compatible custom `inference.py`.
* Learned the correct SageMaker deployment path for staying within budget: use the SDK with `ServerlessInferenceConfig` rather than the one-click UI button, which only offers real-time GPU endpoints.
* Set up AWS Budget Alerts and CloudWatch billing/Free Tier alerts to keep the project's AWS spend under control.
* Correctly handled a git merge conflict on shared code by aborting rather than force-resolving, and deferred the architectural decision on unifying `search.py` to the team.
* Noted that the SageMaker deployment for the embedding model is still **blocked pending an S3 bucket** from the teammate who owns AWS account administration, since Cloudflare R2 (the team's video storage) is incompatible with SageMaker's artifact-fetching requirements; Whisper and BLIP endpoints remain to be deployed after that.
* Completed end-to-end local testing of the search pipeline and finished the remaining sections of the internship report (blogs, self-evaluation, feedback).
