# 06 · Further Resources

[← Debugging LocalStack](./05-debugging.md) | [← Back to README](../README.md)

---

## Table of Contents

- [Official Documentation](#official-documentation)
- [Community & Support](#community--support)
- [Tools & Integrations](#tools--integrations)
- [Learning Resources](#learning-resources)
- [Related Projects](#related-projects)
- [Cheat Sheet](#cheat-sheet)

---

## Official Documentation

| Resource | Link |
|----------|------|
| LocalStack Docs | [docs.localstack.cloud](https://docs.localstack.cloud) |
| Service Coverage Matrix | [docs.localstack.cloud/references/coverage](https://docs.localstack.cloud/references/coverage/) |
| LocalStack GitHub | [github.com/localstack/localstack](https://github.com/localstack/localstack) |
| LocalStack Cloud (Pro) | [localstack.cloud](https://localstack.cloud) |
| Changelog | [github.com/localstack/localstack/releases](https://github.com/localstack/localstack/releases) |
| Docker Hub Image | [hub.docker.com/r/localstack/localstack](https://hub.docker.com/r/localstack/localstack) |

---

## Community & Support

| Resource | Link |
|----------|------|
| LocalStack Slack Community | [localstack.cloud/community](https://localstack.cloud/community) |
| GitHub Discussions | [github.com/localstack/localstack/discussions](https://github.com/localstack/localstack/discussions) |
| GitHub Issues | [github.com/localstack/localstack/issues](https://github.com/localstack/localstack/issues) |
| Stack Overflow tag | [`localstack`](https://stackoverflow.com/questions/tagged/localstack) |
| Twitter / X | [@localstack](https://twitter.com/localstack) |

---

## Tools & Integrations

### CLI Tools

| Tool | Description | Install |
|------|-------------|---------|
| `awslocal` | AWS CLI wrapper for LocalStack | `pipx install awscli-local` |
| `tflocal` | Terraform wrapper for LocalStack | `pip install terraform-local` |
| `cdklocal` | AWS CDK wrapper for LocalStack | `npm install -g aws-cdk-local` |
| `samlocal` | AWS SAM CLI wrapper for LocalStack | `pip install aws-sam-cli-local` |

### Framework Integrations

| Framework | Integration |
|-----------|-------------|
| **Spring Boot** | [Spring Cloud AWS](https://awspring.io/) with profile-based endpoint config |
| **Testcontainers (Java)** | `org.testcontainers:localstack` module |
| **Testcontainers (Python)** | `testcontainers[localstack]` package |
| **Testcontainers (Go)** | `github.com/testcontainers/testcontainers-go/modules/localstack` |
| **Pulumi** | LocalStack provider available |
| **AWS CDK** | Use `cdklocal` CLI wrapper |
| **Serverless Framework** | `serverless-localstack` plugin |
| **Jest (Node.js)** | `jest-localstack-preset` |

### IDE Plugins

| IDE | Plugin |
|-----|--------|
| IntelliJ IDEA | Docker plugin (bundled) + AWS Toolkit |
| VS Code | AWS Toolkit extension |
| VS Code | LocalStack extension (community) |

---

## Learning Resources

### Tutorials & Guides

- [LocalStack Getting Started Guide](https://docs.localstack.cloud/getting-started/)
- [Testing AWS Services Locally with LocalStack and Spring Boot](https://reflectoring.io/spring-cloud-aws-localstack/)
- [Terraform Local Development with LocalStack](https://docs.localstack.cloud/user-guide/integrations/terraform/)
- [CI/CD with LocalStack and GitHub Actions](https://docs.localstack.cloud/user-guide/ci/github-actions/)
- [LocalStack with Testcontainers](https://testcontainers.com/guides/localstack-integration/)

### Videos

- LocalStack YouTube Channel — walkthroughs, demos, and conference talks
- AWS re:Invent talks on local development workflows

### Books & Articles

- *"Cloud Native Patterns"* by Cornelia Davis — local-first development chapter
- Martin Fowler's blog on [Test Double](https://martinfowler.com/bliki/TestDouble.html) — understanding mocks vs emulation

---

## Related Projects

| Project | Description |
|---------|-------------|
| [Moto](https://github.com/getmoto/moto) | Python-based AWS mock library (great for unit tests) |
| [Testcontainers](https://testcontainers.com/) | Spin up real services in Docker for integration tests |
| [AWS SAM](https://aws.amazon.com/serverless/sam/) | Local Lambda development and testing |
| [MinIO](https://min.io/) | S3-compatible object storage server |
| [DynamoDB Local](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.html) | Official AWS local DynamoDB image |
| [ElasticMQ](https://github.com/softwaremill/elasticmq) | SQS-compatible message queue |

> **When to use alternatives?** Use **Moto** when you want lightweight unit-test mocks in Python without Docker. Use **DynamoDB Local** or **MinIO** if you only need a single service and want minimal overhead.

---

## Cheat Sheet

### Quickstart

```bash
# Start LocalStack
docker-compose up -d

# Health check
curl http://localhost:4566/_localstack/health

# Reset all state
curl -X POST http://localhost:4566/_localstack/state/reset

# Stop LocalStack
docker-compose down
```

### Common `awslocal` Commands

```bash
# S3
awslocal s3 mb s3://bucket-name
awslocal s3 ls
awslocal s3 cp file.txt s3://bucket-name/
awslocal s3 rb s3://bucket-name --force

# DynamoDB
awslocal dynamodb list-tables
awslocal dynamodb scan --table-name MyTable

# SQS
awslocal sqs list-queues
awslocal sqs create-queue --queue-name my-queue

# Lambda
awslocal lambda list-functions
awslocal lambda invoke --function-name my-fn --payload '{}' out.json

# SNS
awslocal sns list-topics
awslocal sns create-topic --name my-topic
```

### Environment Variables Reference

```bash
SERVICES=s3,dynamodb,sqs,sns,lambda   # Enable specific services
DEBUG=1                                # Verbose logging
DATA_DIR=/tmp/localstack/data          # Persist data
AWS_DEFAULT_REGION=us-east-1
AWS_ACCESS_KEY_ID=test
AWS_SECRET_ACCESS_KEY=test
```

### Profile Switching

```bash
# Local (LocalStack)
SPRING_PROFILES_ACTIVE=local mvn spring-boot:run

# Production (real AWS)
SPRING_PROFILES_ACTIVE=prod mvn spring-boot:run
```

---

> Found this guide helpful? Consider contributing improvements via a pull request or opening an issue to suggest new content!

---

[← Debugging LocalStack](./05-debugging.md) | [← Back to README](../README.md)
