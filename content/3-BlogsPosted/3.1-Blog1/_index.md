---
title: "Blog 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# BIG BANG, ROLLING, BLUE/GREEN OR CANARY — WHICH DEPLOYMENT STRATEGY SHOULD YOU CHOOSE ON AWS?

Hi everyone!

After some time working with AWS, I learned that there are 4 commonly used deployment strategies: **Big Bang**, **Rolling**, **Blue/Green**, and **Canary**. Choosing the right strategy for your project is essential to minimize risk, reduce downtime, and maintain user trust.

## 1. Big Bang

* **How it works:** stop the old system → deploy everything → switch on the new version.
* **Pros:** simple, no need to maintain two versions in parallel, low cost since only one environment is needed.
* **Cons:** causes downtime; if an error occurs, all users are affected, and a rollback plan must be ready in advance.

## 2. Rolling

* **How it works:** update a small group of instances/pods, wait for a healthy health check, then move on to the next group — gradually updating the entire system.
* **Pros:** almost no downtime; if an error occurs, only a small portion is affected.
* **Cons:** rollout takes longer, and while it's in progress, two versions run in parallel → can easily cause compatibility issues (old/new API, message format in queues, DB schema…).

## 3. Blue/Green

* **How it works:** maintain two identical environments. Blue currently serves users while you deploy the new version to Green, test it freely, and switch traffic to Green once satisfied. Blue is kept as a backup.
* **Pros:** near-zero downtime; rollback is just switching traffic back, taking only seconds.
* **Cons:** doubles infrastructure cost while running in parallel, and requires data + configuration parity between both environments.

## 4. Canary

* **How it works:** release to a very small portion of users or infrastructure first (1%), monitor real signals, then gradually increase to 5% → 25% → 100%.
* **Pros:** detects issues early, minimizes impact on users, increases release agility.
* **Cons:** requires solid monitoring and traffic-routing capability.

## So which one should you choose?

| Strategy | Downtime | Rollback | Cost | Best for |
|---|---|---|---|---|
| Big Bang | High | Hard | Low | Simple systems, infrequent releases, downtime acceptable |
| Rolling | Low | Medium | Low | A balanced starting point for most teams |
| Blue/Green | Near-zero | Easiest | Temporarily doubled | Mission-critical systems |
| Canary | Minimal | Easy | Medium | Continuous CI/CD, high-traffic systems |

There is no single "best" strategy. Choose the tactic that fits your system's complexity, your tolerance for downtime, and your acceptable level of risk.

🔗 **Original post on AWS Study Group:** [View on Facebook](https://www.facebook.com/groups/awsstudygroupfcj/posts/2224001141698179)
