# LocalStack Guide

> A comprehensive, developer-friendly guide to setting up, using, and mastering LocalStack for local AWS development.

[![LocalStack](https://img.shields.io/badge/LocalStack-v3.x-6B3FD6?logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCI+PHBhdGggZmlsbD0id2hpdGUiIGQ9Ik0xMiAyQzYuNDggMiAyIDYuNDggMiAxMnM0LjQ4IDEwIDEwIDEwIDEwLTQuNDggMTAtMTBTMTcuNTIgMiAxMiAyeiIvPjwvc3ZnPg==)](https://localstack.cloud)
[![AWS CLI](https://img.shields.io/badge/AWS_CLI-v2-FF9900?logo=amazonaws)](https://aws.amazon.com/cli/)
[![Docker](https://img.shields.io/badge/Docker-required-2496ED?logo=docker)](https://www.docker.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## What is this guide?

This guide walks you through everything you need to know to use **LocalStack** effectively in your day-to-day development workflow. Whether you're a backend developer tired of incurring AWS costs during testing, a DevOps engineer building IaC pipelines, or a team lead standardizing local environments ,this guide is for you.

---

## Why LocalStack?

| Without LocalStack | With LocalStack |
|--------------------|-----------------|
| AWS costs even for dev/test | Zero cost local emulation |
| Requires internet connection | Fully offline capable |
| Risk of touching prod resources | Safe isolated environment |
| Latency to remote AWS regions | Near-instant local responses |
| Needs real IAM credentials | Dummy credentials work fine |

---

## Table of Contents

| # | Section | Description |
|---|---------|-------------|
| 1 | [Introduction](./docs/01-introduction.md) | What LocalStack is, how it works, and when to use it |
| 2 | [Installation & Setup](./docs/02-installation.md) | Step-by-step setup for WSL2, Docker, AWS CLI, and LocalStack |
| 3 | [Core Concepts](./docs/03-core-concepts.md) | Architecture, service endpoints, profiles, and IaC integration |
| 4 | [Usage Examples](./docs/04-usage-examples.md) | CLI commands, Spring Boot, Terraform, and shell scripts |
| 5 | [Debugging LocalStack](./docs/05-debugging.md) | Logs, common errors, tips, and the LocalStack dashboard |
| 6 | [Further Resources](./docs/06-resources.md) | Official docs, community links, and tools |

---

## Quick Start (TL;DR)

If you just want to get LocalStack running immediately:

```bash
# 1. Start LocalStack via Docker
docker run --rm -it -p 4566:4566 localstack/localstack

# 2. Create an S3 bucket
aws --endpoint-url=http://localhost:4566 s3 mb s3://my-test-bucket

# 3. List buckets
aws --endpoint-url=http://localhost:4566 s3 ls
```

> For a full setup including Docker Compose, Terraform, and Spring Boot integration, see [Installation & Setup](./docs/02-installation.md).

---

## Repository Structure

```
localstack-guide/
├── README.md                  ← You are here
└── docs/
    ├── 01-introduction.md     ← Concepts & internals
    ├── 02-installation.md     ← Full setup guide
    ├── 03-core-concepts.md    ← Architecture & patterns
    ├── 04-usage-examples.md   ← Hands-on examples
    ├── 05-debugging.md        ← Troubleshooting
    └── 06-resources.md        ← Links & references
```

---

## Services Covered

LocalStack supports a wide range of AWS services. This guide focuses on the most commonly used ones:

`S3` · `DynamoDB` · `SQS` · `SNS` · `Lambda` · `IAM` · `KMS` · `ElastiCache` · `EC2` · `EKS` · `ELBv2` · `Route53` · `CloudFront`


---

## Contributing

Feel free to open issues or pull requests if you spot something outdated or want to add more examples. All contributions are welcome!

---

> **Tip:** Star this repo to keep it handy the next time you're setting up a local AWS environment!
