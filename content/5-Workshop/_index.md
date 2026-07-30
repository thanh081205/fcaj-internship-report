---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Deploying VideoPlatformServer on AWS ECS Fargate

#### Overview

**VideoPlatformServer** is a video sharing platform built from two independent services: `api_service` (NestJS + Apollo GraphQL) handles authentication, video management, and asynchronous transcoding, while `search_service` (Python FastAPI) provides hybrid search that combines keyword matching with semantic vector similarity through Qdrant. The two services communicate over bidirectional gRPC and exchange metadata updates asynchronously through RabbitMQ.

In this workshop, you will deploy that system end to end on AWS. Rather than a single-service demo, this walkthrough covers the problems that appear in real production workloads: running containers on **ECS Fargate**, connecting them to managed data stores inside a private VPC, handling asynchronous jobs and inter-service messaging, exposing the platform securely through a CDN, and automating deployments with CI/CD.

A key design decision throughout this workshop is that **all compute runs in private subnets without a NAT Gateway**. Instead, ECS tasks reach AWS services through **VPC Endpoints**, which keeps traffic off the public Internet.

#### What you will build

+ A **VPC** with public and private subnets, security groups, and VPC Endpoints for ECR, CloudWatch Logs, Systems Manager, and Amazon MQ.
+ A managed data layer: **Amazon RDS for PostgreSQL**, **Amazon ElastiCache for Valkey**, **Amazon MQ (RabbitMQ)**, and **Amazon EFS**.
+ Three services on **Amazon ECS Fargate**: `api-service`, `search-service`, and a self-hosted **Qdrant** vector database backed by EFS.
+ Public access through an **Application Load Balancer** and **Amazon CloudFront**, secured with a custom domain (Route 53) and TLS certificates (AWS Certificate Manager).
+ An automated **CI/CD pipeline** using GitHub Actions with an IAM OIDC federated role, so no static AWS access keys are stored anywhere.

#### Content

1. [Introduction](5.1-Workshop-overview/)
2. [Prerequisites](5.2-Prerequisites/)
3. [Network foundation](5.3-Network/)
4. [Data layer](5.4-Data-layer/)
5. [Deploy services to ECS Fargate](5.5-ECS-deployment/)
6. [Public access with ALB and CloudFront](5.6-Public-access/)
7. [CI/CD with GitHub Actions](5.7-CICD/)
8. [Clean up](5.8-Cleanup/)
