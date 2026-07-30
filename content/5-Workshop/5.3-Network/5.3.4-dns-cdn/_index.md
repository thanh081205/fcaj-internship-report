---
title : "DNS and the CloudFront edge"
date : 2024-01-01
weight : 4
chapter : false
pre : " <b> 5.3.4 </b> "
---

The VPC handles traffic once it arrives. This page sets up how it arrives in the first place: the public DNS zone, the TLS certificates, and the CloudFront distribution that sits in front of everything.

These steps come early for a practical reason. An ACM certificate is only issued after DNS validation succeeds, and DNS propagation is the one part of this workshop you cannot speed up — delegating a domain to Route 53 can take anywhere from a few minutes to several hours.

#### Create the public hosted zone

Open the **Route 53** console → **Hosted zones** → **Create hosted zone**.

| Setting | Value |
|---|---|
| Domain name | your domain, e.g. `example.com` |
| Type | Public hosted zone |

Once created, the zone contains an `NS` record listing four Route 53 nameservers, and an `SOA` record. Those four nameservers are what makes the zone authoritative.

![hosted zone](/images/5-Workshop/5.3-Network/hosted-zone.png)

If you registered the domain **through Route 53**, delegation is already done and you can skip the next step. If you registered it **elsewhere**, go to your registrar's control panel and replace its default nameservers with the four from this zone. Nothing else in this workshop will work until that change takes effect.

#### Request two TLS certificates

This is the part that catches people out: you need **two certificates for the same domain**, in two different regions.

| Certificate | Region | Used by |
|---|---|---|
| `example.com` + `*.example.com` | **`us-east-1`** | CloudFront |
| `example.com` + `*.example.com` | `ap-southeast-1` | Application Load Balancer |

{{% notice warning %}}
CloudFront only accepts certificates from **US East (N. Virginia) — `us-east-1`**. This is not a preference or a default that can be changed; it is a hard constraint, because CloudFront is a global service whose control plane lives in that region. A certificate issued in `ap-southeast-1` will simply not appear in the CloudFront distribution's certificate dropdown, with no explanation as to why.
{{% /notice %}}

For each certificate: open **AWS Certificate Manager**, confirm the region selector in the top-right corner is correct, then **Request** → **Request a public certificate**.

| Setting | Value |
|---|---|
| Fully qualified domain name | `example.com` |
| Additional name | `*.example.com` |
| Validation method | **DNS validation** |
| Key algorithm | RSA 2048 |

Adding the wildcard `*.example.com` alongside the apex means a single certificate covers `app.example.com`, `api.example.com`, and anything else you add later, so you never have to repeat this step.

Choose **DNS validation** rather than email validation. DNS validation renews automatically for as long as the validation record stays in the zone; email validation requires a human to click a link every time.

#### Validate the certificates

Each certificate opens in `Pending validation` with a `CNAME` record it needs to see in your DNS. Because the hosted zone is in the same account, ACM can write that record for you: open the certificate and choose **Create records in Route 53**.

![certificate 1 issued](/images/5-Workshop/5.3-Network/cert1.png)

![certificate 2 issued](/images/5-Workshop/5.3-Network/cert2.png)

#### How the edge is designed

CloudFront is the only thing the public actually connects to. The Application Load Balancer has a public DNS name, but no user is ever pointed at it — every request enters through CloudFront, which forwards to the ALB or to S3 depending on the path.

The distribution has **two origins**:

| Origin | Points at | Serves |
|---|---|---|
| ALB origin | the ALB's DNS name | GraphQL, REST, and search requests |
| S3 origin | the video bucket | Video files and DASH segments |

![cloudfront origins](/images/5-Workshop/5.3-Network/cloudfront-origin.png)

and **three behaviors**, evaluated in order:

| Precedence | Path pattern | Origin | Cache policy | Why |
|---|---|---|---|---|
| 0 | `/public/*` | S3 | CachingOptimized | Public video segments — highly cacheable |
| 1 | `/private/*` | S3 | CachingOptimized | Access-controlled media |
| 2 | `Default (*)` | ALB | CachingDisabled | API traffic — must never be cached |

![cloudfront behaviors](/images/5-Workshop/5.3-Network/cloudfront-behaviors.png)

Getting the default behavior right matters more than it looks. If the default cache policy caches responses, CloudFront will happily serve one user's GraphQL response to another user, because the `Authorization` header is not part of the cache key unless you explicitly add it. Use `CachingDisabled` on the default behavior and forward all headers, cookies, and query strings to the origin.

The reason for putting CloudFront in front of the ALB at all:

+ **Video delivery is the bulk of the traffic.** DASH segments are static files requested many times. Serving them from edge locations rather than from `ap-southeast-1` is both faster for viewers and cheaper than paying ALB and S3 egress for every request.
+ **TLS terminates at the edge**, close to the user, which cuts the handshake round-trips on a first connection.
+ **The ALB is shielded.** Only CloudFront's IP ranges need to reach it, and AWS WAF can be attached at the edge later without touching the application.

#### The DNS records you will add later

Once the distribution exists in section 5.6, the zone gets one more record:

| Name | Type | Value |
|---|---|---|
| `app.example.com` | A — Alias | the CloudFront distribution |

Use an **alias** record, not a `CNAME`. Alias records are a Route 53 feature that resolve to the AWS resource's current IP addresses, are free to query, and — unlike `CNAME` — are legal at a zone apex if you later want `example.com` itself to point at CloudFront.

#### A second, private zone

Route 53 also handles name resolution *inside* the VPC. In section 5.5 the ECS services register into a private hosted zone, `vsp.internal`, through AWS Cloud Map. That gives each task a stable internal name such as `api.vsp.internal`, so the two services can find each other over gRPC without hardcoding IP addresses that change on every deployment.

That zone is created automatically when the Cloud Map namespace is set up, so there is nothing to do here — but it is worth knowing that the two zones exist for entirely different purposes: the public zone routes users to the edge, the private zone lets containers find each other.

![private zone](/images/5-Workshop/5.3-Network/vsp-internal-zone.png)
