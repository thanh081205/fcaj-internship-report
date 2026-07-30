---
title: "Event 2"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 4.2. </b> "
---

# Summary Report: "AWS Study Group – FCAJ Tech Sharing Session"

**Date & Time:** July 11, 2026

**Location:** 26th Floor, Bitexco Tower, 02 Hai Trieu Street, Saigon Ward, Ho Chi Minh City

**Role:** Attendee

### Event Overview

This session was a technical sharing event hosted at the AWS office as part of the First Cloud AI Journey (FCAJ) program, featuring three back-to-back talks from community speakers on certification, security automation, and observability.

### Talk 1 — "Inside The Exam: AWS Cloud Practitioner" (Speaker: Ngo Le Tan Huy)

A strategic roadmap for passing the AWS Certified Cloud Practitioner (CLF-C02) exam:

- **Exam structure:** 65 multiple-choice/multiple-response questions, 90 minutes (+30 minutes for non-native English speakers), passing score 700/1000, valid for 3 years.
- **Domain weighting:** Cloud Concepts (24%), Security and Compliance (30%), Cloud Technology and Services (34%), Billing/Pricing/Support (12%).
- **Preparation strategy:** "map keyword thinking" (associating services with real-world use-case keywords), reviewing wrong answers instead of just doing mock tests, and hands-on practice on the AWS Free Tier.
- **Exam-day tips:** elimination technique, not overthinking foundational-level questions, watching for tricky wording ("not", "least cost", "most scalable"), and preparing ID documents for Pearson VUE test centers.

### Talk 2 — "Securing Your Web Apps With AWS Security Agent" (Speaker: Nguyen Tuan Thinh)

An introduction to AWS's autonomous security agent, powered by Amazon Bedrock:

- **Problem addressed:** manual pentests are slow, expensive ($5k–$20k per engagement), and inconsistent.
- **Capabilities:** covers the full security lifecycle — Design Review (architecture docs vs. PCI DSS/NIST CSF/AWS Well-Architected), Code Review (auto-scans PRs on GitHub/GitLab, suggests fixes), and automated Penetration Testing (multi-step exploit chains, e.g., IDOR → XSS, with verifiable proof of exploitation).
- **Pricing:** pay-as-you-go per task-hour (~$50/hour), free trial with 400 task-hours over 2 months; a real case study cost $1,500–$2,500 in agent work versus a traditional pentest team.
- **Limitations:** blocked by strong auth (MFA/biometrics/mTLS), struggles with business-logic flaws, and requires monitoring to control task-hour consumption on complex apps.

### Talk 3 — "SLA and Monitoring: From SLA to Monitoring, What Really Matters" (Speaker: Nguyen Huynh Son)

A talk on why infrastructure health metrics alone don't guarantee a good user experience, illustrated with a live demo:

- **Core message:** *Healthy Infrastructure ≠ Healthy User Experience.* AWS's SLA guarantees the cloud; the customer experience is the team's own responsibility.
- **Monitoring pyramid:** Cloud Provider → Infrastructure → Application → Business → Customer Experience. Lower layers help diagnose root cause; upper layers tell you if users/business are actually impacted.
- **Live demo:** a 3-tier app (User → ALB → EC2 → RDS) kept showing a fully "green" dashboard (CPU 18%, ALB target healthy, `/health` returning 200 OK) even after the demo broke the security group between EC2 and RDS — because the health check endpoint never touches the database, while the real `/login` path does. Login success rate silently dropped from 100% to 0% with no infrastructure alarm firing.
- **Takeaway on alerting:** a proper flow needs custom business metrics (e.g., login failure rate) feeding CloudWatch Alarms → SNS → Email/Slack, so the team is notified before customers complain.

### Key Takeaways

- Certification prep benefits from pattern recognition (keyword-to-service mapping) more than rote memorization.
- Security testing is shifting toward AI agents that can autonomously chain and verify exploits, changing the cost/speed trade-off compared to traditional pentesting.
- Infrastructure-level monitoring (CPU, memory, health checks) is not sufficient on its own — teams need business/user-journey metrics (e.g., login or checkout success rate) to know when users are actually affected.

### Event Photos

![Audience at the AWS office during the sharing session](/images/4-EventParticipated/4.2-Event2/audience-overview.jpg)
*Full house at the AWS office for the FCAJ tech sharing session*

![Live AWS certification quiz on screen during the session](/images/4-EventParticipated/4.2-Event2/certification-quiz.jpg)
*Interactive AWS certification quiz between talks*

![Team scoreboard game at the event](/images/4-EventParticipated/4.2-Event2/team-scoreboard-game.jpg)
*Team scoreboard activity during the session*
