---
title : "Create the VPC, subnets, and routing"
date : 2024-01-01
weight : 1
chapter : false
pre : " <b> 5.3.1 </b> "
---

In this step we create the VPC, three subnets, an Internet Gateway, and the two route tables that decide which subnets can reach the Internet.

#### Create the VPC and subnets

Open the **VPC** console and choose **Create VPC**.

![create vpc](/images/5-Workshop/5.3-Network/create-vpc.png)

Set the following:

| Setting | Value |
|---|---|
| IPv4 CIDR block | `10.0.0.0/16` |
| IPv6 CIDR block | No IPv6 CIDR block |
| Tenancy | Default |
| DNS hostnames | Enabled |
| DNS resolution | Enabled |

**DNS hostnames** and **DNS resolution** must both be enabled. VPC Endpoints rely on Private DNS to override the public AWS service names with private IP addresses, and Private DNS silently refuses to work if either setting is off.

Choose **Create VPC** and wait for it to finish.

Now create the subnets: two public subnets, `10.0.1.0/24` and `10.0.2.0/24`, and one private subnet, `10.0.10.0/24`.

![subnet 1](/images/5-Workshop/5.3-Network/subnet1.png)

![subnet 2](/images/5-Workshop/5.3-Network/subnet2.png)

![subnet 3](/images/5-Workshop/5.3-Network/subnet3.png)

#### Create an Internet Gateway for the VPC

![igw](/images/5-Workshop/5.3-Network/igw.png)

#### Create the route tables

Create a route table for the public subnets, with a route pointing at the Internet Gateway you just created.

![public route table](/images/5-Workshop/5.3-Network/public-rt.png)

Create a route table for the private subnet, with a route pointing at the endpoint used to reach **S3**.

![private route table](/images/5-Workshop/5.3-Network/private-rt.png)
