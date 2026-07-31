---
title: "Blog 3"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---
# AMAZON BEDROCK MANAGED KNOWLEDGE BASE — "MANAGED" RAG FOR ENTERPRISE, NO NEED TO BUILD YOUR OWN PIPELINE

Hi everyone!

AWS just announced (17/06/2026) a new Bedrock feature that I think is quite notable for anyone working on agentic AI / RAG: **Amazon Bedrock Managed Knowledge Base**. Here's a summary of the key points.

## 1. The problem it solves

When building a knowledge base for an agent from scratch, developers typically run into 3 difficulties:

* Connecting enterprise data: data is scattered across many systems (SharePoint, Confluence, Drive...), each with its own access control and format → you have to write custom connectors for each one.
* Optimizing RAG accuracy: you have to keep experimenting with parsing strategy, chunking, embedding model... to get accurate, sufficiently-contextualized answers.
* Operating infrastructure at scale: millions of documents or thousands of small knowledge bases all need stable, secure infrastructure with cost control.

## 2. Three main features

* **Native data connectors**: 6 built-in connectors (Amazon S3, SharePoint, Confluence, Web Crawler, Google Drive, OneDrive) that automatically pull in data and access permissions from SaaS applications.
* **Smart Parsing**: automatically picks the right parsing strategy for each data type — preserving HTML/table/image structure when crawling the web, preserving document hierarchy when pulling from SharePoint, and auto-detecting bounding boxes to extract and caption images/video.
* **Agentic Retriever**: handles complex questions requiring multi-hop reasoning — automatically plans a step-by-step query, looks things up across multiple knowledge bases, then synthesizes an answer.

## 3. Integration with AgentCore Gateway

Managed Knowledge Base is available out of the box as a pre-built target type in Amazon Bedrock AgentCore Gateway, automatically exposed via the MCP standard, so frameworks like Strands Agents, LangChain, CrewAI, LlamaIndex, and LangGraph can use it immediately without custom integration code.

## 4. Not locked into a fixed model

Managed Knowledge Base decouples the infrastructure (connectors, parsing, storage, retrieval) from model selection, so you can always use the latest model, picking a cheap/fast model for simple questions and a more powerful model for complex ones on the same infrastructure. If you're already using the older Bedrock Knowledge Bases API, migration doesn't require any code changes.

## 5. Pricing & regions

Billed based on indexed data volume and number of retrievals (on-demand), with no upfront commitment. Currently available in US East (N. Virginia), US West (Oregon), Asia Pacific (Sydney, Tokyo), Europe (Dublin, Frankfurt, London), and AWS GovCloud (US-West).

## Key takeaway

Amazon Bedrock Managed Knowledge Base turns the entire RAG pipeline (connector + parsing + embedding + retrieval + re-ranking) into a single managed primitive, letting developers focus on business logic instead of building infrastructure from scratch.

Source: [aws.amazon.com/blogs/aws/introducing-amazon-bedrock-managed-knowledge-base](https://aws.amazon.com/blogs/aws/introducing-amazon-bedrock-managed-knowledge-base-for-faster-more-accurate-enterprise-ai-applications/)

🔗 **Original post on AWS Study Group:** [View on Facebook](https://www.facebook.com/groups/awsstudygroupfcj/posts/2226890134742613)