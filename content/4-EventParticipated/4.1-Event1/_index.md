---
title: "Event 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 4.1. </b> "
---

# Summary Report: "First Cloud Journey (FCJ) Community Day"

**Date & Time:** July 11, 2026

**Location:** 26th Floor, Bitexco Tower, 02 Hai Trieu Street, Saigon Ward, Ho Chi Minh City

**Role:** Attendee

### Event Overview

Community Day was a multi-track sharing event at the AWS office, part of the First Cloud Journey (FCJ) community, featuring six back-to-back talks from FCJ members and alumni covering security, DevOps fundamentals, career growth, game networking, teamwork, and generative AI.

### Talk 1 — "WAF + ML for Cyber Attack Detection" (Speaker: Le Hoang Gia Dai)

Building a Machine Learning-based Network Intrusion Detection System (NIDS) to complement AWS WAF:

- **Why WAF alone isn't enough:** AWS WAF's rule-based detection handles known attack patterns well but struggles with zero-day, hybrid/spoofing, and novel anomalous behaviors.
- **NIDS approach:** trained a model on the CSE-CIC-IDS2018 dataset (UNB/CIC), covering attack labels like DoS, brute force, SQL injection, and DDoS; used LightGBM, evaluated with a confusion matrix, and addressed class imbalance to improve minority-attack detection.
- **AWS architecture:** VPC, EC2, ALB, WAF, S3, Kinesis Data Firehose, Lambda, Security Hub, GuardDuty, Inspector, SNS, IAM, Config, and CloudWatch working together, with NIDS predictions correlated against WAF events on a real-time dashboard.
- **Lessons learned:** data quality is critical for ML performance; signature-based protection alone is insufficient; ML-based NIDS + AWS WAF gives layered, adaptive defense.

### Talk 2 — "Docker – A Containerization Technology" (Speaker: Bao Huynh)

A beginner-friendly introduction to containerization:

- **Virtualization vs. containerization:** VMs each carry a full OS (heavy, slow to update); containers package an app with just what it needs, staying lightweight and consistent across machines.
- **Docker basics:** images, containers, Dockerfiles, and layer caching (an unchanged layer is reused; a changed layer triggers a rebuild of that layer and everything after it).
- **Use cases:** CI/CD pipelines, microservices, dev/test environments, cloud-native apps, and legacy application modernization.

### Talk 3 — "From IT Helpdesk to Senior Sysadmin" (Speaker: Tran Trung Vinh)

A career-journey talk for students and junior IT staff:

- **Path:** started in IT Helpdesk with no special advantages, built skills in Linux/networking, hands-on labs, and troubleshooting under pressure before moving into a full Sysadmin role.
- **Life as a Sysadmin:** server provisioning, network management, patching, capacity planning — with a key rule of "never test in production."
- **Transition to Cloud/DevOps:** moving from on-premise, manual scaling to a cloud mindset (AWS, elastic scaling, managed services), Infrastructure as Code (Terraform), and DevOps culture (CI/CD, Docker).
- **Career advice:** go deep on 1–2 core skills before spreading out, build a real portfolio (it matters more than certifications alone), and keep going regardless of where you start.

### Talk 4 — "Multiplayer in the Cloud: Connecting Godot Clients with AWS WebSockets" (Speaker: Nguyen Quoc Bao)

Building real-time multiplayer using AWS serverless services and the Godot game engine:

- **Choosing an architecture:** compared UDP/ENet (lowest latency, best for FPS/racing), WebSocket (full-duplex, reliable, best for turn-based/lobby/chat), and HTTP Polling (simple but high latency) — WebSocket was chosen for a turn-based game.
- **AWS architecture:** API Gateway WebSocket API (`$connect`/`$disconnect`/`$default` routes) → Lambda (Node.js) → DynamoDB (`connectionId` as partition key, tracking match state) → CloudWatch for logs.
- **Godot client integration:** `WebSocketPeer` for connecting, polling every frame, and sending/receiving JSON messages to drive matchmaking and game state.
- **Challenges:** stale connections causing `GoneException`, costly full-table DynamoDB scans for matchmaking, and Lambda's stateless nature requiring all game state to round-trip through DynamoDB.
- **What's next:** AWS GameLift for games that need continuous, high-frequency synchronization (vs. WebSocket + Lambda, which fits turn-based/lobby use cases well).

### Talk 5 — "The Art of Effective Teamwork" (Speaker: Truong Huy Phuoc)

A soft-skills talk on working effectively in teams:

- **4 Golden Rules:** clear & shared goals, right person in the right place, open communication & active listening, and personal accountability.
- **Digital tools for teamwork:** ClickUp, Trello, Slack, Google Workspace, and Discord for coordination and communication.

### Talk 6 — "GraphRAG: Building GraphRAG Applications with Amazon Bedrock and Amazon Neptune" (Speaker: Viet Phat)

Going beyond standard RAG for questions that require multi-hop reasoning:

- **RAG's limitation:** a standard RAG pipeline struggles with questions that require chaining facts across multiple entities/documents (e.g., "Where is the company headquartered that was acquired by the company founded by Jeff Bezos?").
- **GraphRAG's answer:** stores relationships explicitly as graph edges and uses graph traversal for multi-hop reasoning instead of pure vector similarity.
- **Two implementation routes on AWS:**
  - *Fully managed:* Amazon Bedrock Knowledge Bases (chunking, entity extraction, embeddings) + Amazon Neptune Analytics (graph storage, relationship discovery).
  - *Custom:* a LlamaIndex pipeline for data prep/knowledge-graph construction + Amazon Neptune for storage, multi-hop traversal, and Cypher queries.

### Key Takeaways

- Layered security (rule-based WAF + ML-based anomaly detection) catches threats that either approach would miss alone.
- Containerization solves the "works on my machine" problem that heavier VM-based virtualization struggles with.
- A strong portfolio of real, hands-on projects matters more for career growth than credentials alone — and "never test in production" is a universal operational rule.
- Choosing the right networking architecture (UDP, WebSocket, or HTTP polling) depends entirely on the real-time requirements of the use case, not a single "best" default.
- Effective teamwork depends on shared goals, the right person for the right task, and personal accountability — not just the tools used to coordinate.
- GraphRAG extends standard RAG with explicit relationship modeling, enabling multi-hop reasoning that flat vector retrieval cannot support.

### Event Photos

![Slide on Network Intrusion Detection System (NIDS) during the WAF + ML talk](/fcaj-internship-report/images/4-EventParticipated/4.1-Event1/nids-talk.jpg)
*NIDS architecture slide from the "WAF + ML for Cyber Attack Detection" talk*

![Slide comparing UDP/ENet, WebSocket, and HTTP Polling architectures](/fcaj-internship-report/images/4-EventParticipated/4.1-Event1/godot-websocket-talk.jpg)
*Choosing a multiplayer networking architecture, from the Godot + AWS WebSocket talk*

![Docker talk disclaimer slide with a friendly intro tone](/fcaj-internship-report/images/4-EventParticipated/4.1-Event1/docker-talk.jpg)
*Kicking off the Docker containerization talk*
