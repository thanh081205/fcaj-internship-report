---
title: "Sharing and Feedback"
date: 2024-01-01
weight: 7
chapter: false
pre: " <b> 7. </b> "
---

### Overall Evaluation

**1. Working Environment**
The AWS office space (26th Floor, Bitexco Tower) is comfortable and well set up for focused work, and it was also the venue for community sessions like Community Day, which made the internship feel connected to a wider builder community rather than just a desk job.

**2. Support from Mentor / Team Admin**
My mentor gave me the space to own the semantic search feature end to end — including letting me work through technical decisions and even a merge conflict with a teammate's branch on my own — while still being available to unblock me when I was stuck (e.g., discussing the SageMaker Serverless Inference deployment path). The admin team handled office access and event logistics smoothly.

**3. Relevance of Work to Academic Major**
Building the `search_service` pipeline (transcription, embeddings, vector search, benchmarking) connected directly to my AI specialization, while the AWS deployment side (SageMaker, IAM, budgets, CloudWatch) pushed me into infrastructure and MLOps territory I hadn't covered much in school.

**4. Learning & Skill Development Opportunities**
Beyond the technical pipeline itself, I learned to benchmark retrieval strategies rigorously (Precision@3, Recall@3, MRR) instead of trusting intuition, to communicate findings to a team through a written report, and to navigate a real cross-team dependency (waiting on a teammate's S3 bucket to unblock deployment) without it stalling my own progress.

**5. Company Culture & Team Spirit**
The FCAJ community felt collaborative rather than competitive — events like the Community Day sharing session gave FCJ members a platform to teach each other, and the willingness of previous cohorts to write things like the FCJ study roadmap made ramping up much easier.

**6. Internship Policies / Benefits**
Flexible access to the AWS office for events and clear registration/approval process (via the FCAJ portal) made it easy to plan around both my coursework and the internship.

---

### Additional Questions
- **Most satisfying moment:** seeing the dense-vs-hybrid benchmark numbers confirm a real, non-obvious engineering decision (dense-only over hybrid RRF) rather than just shipping a default configuration.
- **What could improve:** clearer upfront guidance on shared infrastructure ownership (e.g., who owns the S3 bucket, account limits like SageMaker Serverless concurrency) so cross-team blockers surface earlier.
- **Would I recommend it:** yes — the combination of a real open-ended technical project, a supportive mentor, and an active community (events, blogs, study group) is hard to find in a short internship.

---

### Suggestions & Expectations
- A short, shared checklist for AWS account/resource ownership across a team project (who owns S3, budgets, IAM) would help avoid the kind of blocker I hit with SageMaker deployment.
- I'd be interested in continuing to contribute to the AWS Study Group even after the program ends.
- Overall, this internship gave me a much clearer picture of what it takes to take an ML idea from a notebook to a deployed, benchmarked, cost-controlled AWS service — and I'd gladly continue in this direction.
