---
title: "Week 6 Worklog"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.6. </b> "
---
### Week 6 Objectives:

* Learn container orchestration on AWS beyond basic ECS.
* Scope and propose the personal Workshop project: a **semantic search feature** for `VideoPlatformServer`, a team video-sharing/livestream platform (I independently own the `search_service`, Python/FastAPI, while a teammate owns S3/AWS account administration and another owns the `api_service` in NestJS/TypeScript).
* Attend a community tech-sharing session at the AWS office.

### Tasks carried out this week:
| Day | Task | Date | Reference Material |
| --- | --- | --- | --- |
| 1 (Mon) | Containerization with **Amazon ECS and AWS Fargate**: serverless containers | 06/07/2026 | <https://000067.awsstudygroup.com> |
| 2 (Tue) | Getting Started with **Amazon EKS**: clusters, node groups, kubectl | 07/07/2026 | <https://000126.awsstudygroup.com> |
| 3 (Wed) | Draft the Workshop Proposal: semantic search over video content (transcription + visual captioning + embeddings + vector search) | 08/07/2026 | [Proposal](/2-proposal/) |
| 4 (Thu) | Design the `search_service` architecture: RabbitMQ job queue, Qdrant vector database, service boundary with the team's `api_service` | 09/07/2026 | |
| 5 (Fri) | Set up base infrastructure for `search_service`: RabbitMQ, Qdrant, PostgreSQL, Redis via Docker Compose; create a dedicated `video-semantic-indexing` queue separate from the team's NestJS BullMQ `video-processing` queue | 10/07/2026 | |
| 6 (Sat) | Attended **AWS Study Group – FCAJ Tech Sharing Session** at the AWS office (see [Event 1](/4-eventparticipated/4.2-event2/)) | 11/07/2026 | |

### Week 6 Achievements:

* Deployed a container on Amazon ECS with Fargate and stood up a first Amazon EKS cluster for practice.
* Completed the first draft of the Workshop Proposal, scoping a semantic search feature for the team's video platform.
* Designed the `search_service` as an independently owned Python/FastAPI microservice, decoupled from the team's `api_service`.
* Learned the importance of **queue isolation**: a dedicated `video-semantic-indexing` RabbitMQ queue prevents job splitting with the team's existing `video-processing` queue.
* Attended an external tech-sharing session covering AWS certification strategy, the AWS Security Agent, and SLA/monitoring best practices — see the [Events Participated](/4-eventparticipated/) section for details.
