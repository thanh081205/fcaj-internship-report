---
title: "Blog 2"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---

# ZERO-SECRET DEPLOYMENT WITH SECRETS MANAGER + IAM ROLE + IMDSV2 — A 4-HOUR DEBUGGING LESSON OVER ONE NEWLINE CHARACTER

In my first month building CI/CD, I was still copying a `.env` file to the server via `scp` — the most serious mistake of my internship. I found out the hard way after 4 hours of debugging why `FIREBASE_PRIVATE_KEY` kept getting corrupted, because `jq` wasn't escaping newlines correctly. From that point on, I refactored the pipeline to achieve **"zero-secret deployment"**: secrets live only in AWS Secrets Manager, EC2 fetches them via IAM Role + IMDSv2 on every deploy, and they disappear right after the container starts.

## Golden Rules

* Secrets never sit inside a Dockerfile (`COPY`).
* Never commit them to git.
* Never keep a permanent `.env` file on EC2.
* Right after the container starts, `rm -f` the temporary `.env` file immediately.

## IAM Role vs. Access Key

* **IAM User** carries long-term credentials — high risk if leaked.
* **IAM Role** issues temporary credentials via STS, valid for 1 hour, auto-rotating.
* **Setup:** create an `EC2-Backend-Role`, attach the `SecretsManagerReadWrite` policy, and attach it to the EC2 instance — the instance fetches credentials via IMDSv2 automatically, with no access key ever hardcoded.

## The Newline Bug

A real private key contains an actual newline character (`0x0A`), but storing it in a JSON secret requires the escape sequence `\n`. `jq` treated `\n` as plain text instead of an escape sequence → Firebase threw `"Invalid PEM formatted key"`.

The fix: use a Python heredoc (`python3 - <<'PYEOF'` + `json.loads()`) instead of `jq`, since Python correctly follows the JSON spec.

## Revisiting the 2019 Capital One Breach

106 million records were exposed through a combination of an SSRF vulnerability in a WAF and IMDSv1 accepting a simple GET request to retrieve STS credentials. IMDSv2 blocks this with a session-oriented protocol — a PUT request with a token header is required before any GET, and SSRF exploits typically can't send a PUT.

## Enforcing IMDSv2

```
aws ec2 modify-instance-metadata-options --instance-id i-xxx \
  --http-tokens required --http-put-response-hop-limit 3 --http-endpoint enabled
```

You can enforce this account-wide with an SCP that blocks `ec2:RunInstances` when `HttpTokens=required` is missing.

## Rotation & Auditing

* Auto-rotate via Lambda every 90 days (`aws secretsmanager rotate-secret ...`).
* CloudTrail logs every `GetSecretValue` call.
* A CloudWatch Alarm fires if the same instance calls it more than 10 times per hour.

## Key Takeaway

When working with structured data (JSON/YAML/TOML), always use a language with a proper parser (`json.loads`, `yaml.safe_load`) — never `jq`/`sed`/`awk`/`grep`. And from the Capital One breach: layered defense is not optional — skip IMDSv2 and proper IAM Role usage, and one small SSRF vulnerability can turn into a disaster.

🔗 **Original post on AWS Study Group:** [View on Facebook](https://www.facebook.com/groups/awsstudygroupfcj/posts/2226205278144432)
