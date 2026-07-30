---
title : "Introduction"
date : 2024-01-01
weight : 1
chapter : false
pre : " <b> 5.1. </b> "
---

#### Amazon ECS on AWS Fargate

+ **Amazon ECS** is a fully managed container orchestration service. You describe your application with a *task definition* (which image to run, how much CPU and memory it needs, which ports it opens), and ECS keeps the number of copies you asked for running as a *service*.
+ **AWS Fargate** is the serverless compute engine behind ECS. There are no EC2 instances to patch or scale: you declare CPU and memory per task and AWS handles the infrastructure underneath. You pay only for the resources a task requests, for exactly as long as it runs.
+ **Fargate Spot** offers the same compute at a significantly lower price, in exchange for AWS being able to reclaim the capacity after a two-minute warning. It suits stateless services that tolerate a brief restart.
+ Every Fargate task gets its own elastic network interface (ENI) inside your VPC, so it behaves like any other resource on the network: security groups apply to it, and it connects directly to private services such as RDS.

#### VPC Endpoints and why we avoid a NAT Gateway

Containers running in a **private subnet** have no route to the Internet. They still need to reach AWS services, though — to pull images from ECR, write logs to CloudWatch, or read configuration from Systems Manager. There are two common ways to solve this:

+ A **NAT Gateway** gives the private subnet a path out to the Internet. It is simple, but it is billed per hour *and* per gigabyte that passes through it, and the traffic still leaves for the public Internet.
+ **VPC Endpoints** keep the traffic inside the AWS network. An **interface endpoint** (using AWS PrivateLink) places an ENI for the service directly into your subnet, while a **gateway endpoint** adds a route for Amazon S3 to the route table at no cost.

In this workshop we use VPC Endpoints and **create no NAT Gateway at all**. This keeps the workload isolated from the public Internet and makes the traffic paths explicit: every AWS service a task can reach is one you deliberately created an endpoint for.

#### System architecture

The platform consists of two independent application services plus a self-hosted vector database, all running as ECS Fargate services in the same private subnet:

+ **`api-service`** (NestJS + Apollo GraphQL) — handles authentication, video CRUD, generating presigned upload URLs for S3, and running the asynchronous transcoding worker. It listens on port `8080` for HTTP/GraphQL and `50051` for gRPC.
+ **`search-service`** (Python FastAPI) — handles hybrid search and keeps the vector index in sync. It listens on port `8000` for HTTP and `50052` for gRPC.
+ **`qdrant`** — a self-hosted vector database storing video embeddings, with its data persisted on an Amazon EFS file system so the index survives task restarts.

Around them sit the managed services: **Amazon RDS for PostgreSQL** for relational metadata, **Amazon ElastiCache for Valkey** for sessions, caching, and the BullMQ job queue, **Amazon MQ (RabbitMQ)** for asynchronous messaging between the two services, and **Amazon S3** for both raw and transcoded video files.

External traffic enters through **Amazon CloudFront**, which sits in front of an **Application Load Balancer** for the API and serves video files directly from S3. The custom domain comes from **Route 53**, with a TLS certificate from **AWS Certificate Manager**.

#### What comes next

The remaining sections build this architecture from the bottom up: the network first, then the data layer, then the containers running on top, and finally the public entry point and the deployment pipeline. Each section stands on its own, but they are best followed in order, since later steps reference resources created earlier
