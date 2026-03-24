# 03 · Core Concepts

[← Installation & Setup](./02-installation.md) | [Next: Usage Examples →](./04-usage-examples.md)

---

## Table of Contents

- [The Single Gateway Model](#the-single-gateway-model)
- [Service Emulation vs Mocking](#service-emulation-vs-mocking)
- [AWS Profiles: Local vs Production](#aws-profiles-local-vs-production)
- [Infrastructure as Code with LocalStack](#infrastructure-as-code-with-localstack)
- [Persistence and State Management](#persistence-and-state-management)
- [Networking Inside LocalStack](#networking-inside-localstack)
- [Environment Variable Reference](#environment-variable-reference)
- [Developer Workflow Pattern](#developer-workflow-pattern)

---

## The Single Gateway Model

One of LocalStack's most important design decisions is that **all services are accessible through a single port**: `4566`.

In real AWS, each service has its own DNS endpoint:
- `s3.amazonaws.com`
- `dynamodb.us-east-1.amazonaws.com`
- `sqs.us-east-1.amazonaws.com`

In LocalStack, everything goes through:

```
http://localhost:4566
```

LocalStack uses the request path, headers, and body to determine which service to route each call to internally. This simplifies configuration significantly — you only need to override a single endpoint in your application.

```
Your App
  │
  ├── s3.putObject()      ──▶  localhost:4566  ──▶  S3 Handler
  ├── dynamodb.putItem()  ──▶  localhost:4566  ──▶  DynamoDB Handler
  └── sqs.sendMessage()   ──▶  localhost:4566  ──▶  SQS Handler
```

---

## Service Emulation vs Mocking

It's important to understand the distinction between **emulation** and **mocking**:

| | Mocking | LocalStack Emulation |
|---|---------|----------------------|
| **What it does** | Returns hardcoded/fake responses | Runs actual service logic with real behavior |
| **State** | Stateless | Stateful — resources persist during the session |
| **API compatibility** | Partial | High compatibility with real AWS APIs |
| **Use case** | Unit tests with controlled data | Integration tests with realistic behavior |

For example, LocalStack's S3 actually stores objects (in memory or on disk). Its SQS actually queues messages and delivers them to consumers. This makes integration tests much more reliable than simple mocks.

---

## AWS Profiles: Local vs Production

A core pattern for teams using LocalStack is **profile-based switching** — the same codebase runs against LocalStack locally and against real AWS in staging/production, controlled by environment variables or Spring profiles.

### AWS CLI Profiles

```bash
# LocalStack profile (dummy credentials)
aws configure --profile localstack
# Access Key: test | Secret Key: test | Region: us-east-1

# Production profile (real credentials)
aws configure --profile prod
# Access Key: <real> | Secret Key: <real> | Region: us-east-1
```

Use them:

```bash
# LocalStack
aws --profile localstack --endpoint-url=http://localhost:4566 s3 ls

# Production
aws --profile prod s3 ls
```

### Spring Boot Profiles

`application-local.properties`:
```properties
cloud.aws.endpoint=http://localhost:4566
cloud.aws.region.static=us-east-1
cloud.aws.credentials.access-key=test
cloud.aws.credentials.secret-key=test
```

`application-prod.properties`:
```properties
cloud.aws.region.static=us-east-1
# Credentials sourced from IAM role or environment variables
```

Switch at runtime:
```bash
# Local development
SPRING_PROFILES_ACTIVE=local ./mvnw spring-boot:run

# Production
SPRING_PROFILES_ACTIVE=prod ./mvnw spring-boot:run
```

### Node.js / JavaScript

```javascript
const AWS = require('aws-sdk');

const isLocal = process.env.NODE_ENV === 'local';

const s3 = new AWS.S3(isLocal ? {
  endpoint: 'http://localhost:4566',
  accessKeyId: 'test',
  secretAccessKey: 'test',
  s3ForcePathStyle: true,   // required for LocalStack
} : {});
```

### Python (boto3)

```python
import boto3
import os

endpoint = 'http://localhost:4566' if os.getenv('ENV') == 'local' else None

s3 = boto3.client(
    's3',
    endpoint_url=endpoint,
    aws_access_key_id='test',
    aws_secret_access_key='test',
    region_name='us-east-1'
)
```

> 💡 **Best practice:** Never hardcode the LocalStack endpoint. Always read it from an environment variable or config file so switching environments requires zero code changes.

---

## Infrastructure as Code with LocalStack

### Terraform

Terraform works with LocalStack out of the box. The key is overriding the service endpoints in the `provider "aws"` block:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_requesting_account_id  = true
  s3_use_path_style           = true

  endpoints {
    s3          = "http://localhost:4566"
    dynamodb    = "http://localhost:4566"
    sqs         = "http://localhost:4566"
    sns         = "http://localhost:4566"
    iam         = "http://localhost:4566"
    kms         = "http://localhost:4566"
    lambda      = "http://localhost:4566"
    ec2         = "http://localhost:4566"
    route53     = "http://localhost:4566"
    cloudfront  = "http://localhost:4566"
    elbv2       = "http://localhost:4566"
    eks         = "http://localhost:4566"
  }
}
```

Run Terraform against LocalStack:

```bash
terraform init
terraform plan
terraform apply -auto-approve
```

This lets you validate your IaC scripts fully locally before applying them to real AWS. Especially valuable when making large infra changes.

### tflocal (Terraform wrapper for LocalStack)

Similar to `awslocal`, there's a `tflocal` wrapper that handles the endpoint configuration automatically:

```bash
pip install terraform-local

tflocal init
tflocal plan
tflocal apply
```

---

## Persistence and State Management

By default, LocalStack is **ephemeral** — all resources are lost when the container stops. This is fine for most test scenarios.

### Enabling Persistence

Mount a volume in your Docker Compose to persist state across restarts:

```yaml
environment:
  - DATA_DIR=/tmp/localstack/data
volumes:
  - "./localstack-data:/tmp/localstack/data"
```

Or with the `PERSIST` flag (LocalStack Pro):

```yaml
environment:
  - PERSIST=1
```

### Initialisation Scripts

For consistent test setups, use LocalStack's init hook — scripts placed in specific directories are automatically executed on startup:

```
localstack-init/
├── ready.d/
│   └── 01-create-resources.sh   # Runs after LocalStack is ready
└── shutdown.d/
    └── 01-cleanup.sh            # Runs on shutdown
```

Example `01-create-resources.sh`:

```bash
#!/bin/bash
awslocal s3 mb s3://app-assets
awslocal dynamodb create-table \
  --table-name Users \
  --attribute-definitions AttributeName=UserId,AttributeType=S \
  --key-schema AttributeName=UserId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

echo "LocalStack initialized with default resources"
```

Mount it in Docker Compose:

```yaml
volumes:
  - "./localstack-init:/etc/localstack/init"
```

This ensures every developer starts with the same resources — great for onboarding and CI pipelines.

---

## Networking Inside LocalStack

### Path-Style vs Virtual-Hosted-Style S3

Real AWS uses virtual-hosted-style URLs by default:
```
https://my-bucket.s3.amazonaws.com/key
```

LocalStack requires **path-style** S3 access:
```
http://localhost:4566/my-bucket/key
```

Always enable path-style in your AWS SDK configs when using LocalStack:

```java
// Java
.withPathStyleAccessEnabled(true)
```

```python
# Python
s3 = boto3.client('s3', config=Config(s3={'addressing_style': 'path'}))
```

### Lambda Networking

When a Lambda function running inside LocalStack needs to call other LocalStack services, it should use the internal Docker network hostname instead of `localhost`:

```bash
# Inside Lambda — use this instead of localhost:4566
http://localstack:4566
```

Or use the special hostname `host.docker.internal` on Docker Desktop.

---

## Environment Variable Reference

| Variable | Description | Default |
|----------|-------------|---------|
| `SERVICES` | Comma-separated list of services to enable | All |
| `DEBUG` | Enable verbose debug logging (`1` = on) | `0` |
| `DATA_DIR` | Path to persist data inside the container | None |
| `AWS_DEFAULT_REGION` | Default AWS region | `us-east-1` |
| `AWS_ACCESS_KEY_ID` | Dummy access key | `test` |
| `AWS_SECRET_ACCESS_KEY` | Dummy secret key | `test` |
| `LOCALSTACK_API_KEY` | License key for LocalStack Pro | — |
| `LAMBDA_EXECUTOR` | Lambda execution mode (`docker`, `local`) | `docker` |
| `PORT_WEB_UI` | Port for LocalStack web UI | `8080` |

---

## Developer Workflow Pattern

Here's the recommended inner-loop workflow when using LocalStack:

```
1. Start LocalStack
   └── docker-compose up -d

2. Initialize resources (if not using init scripts)
   └── awslocal s3 mb s3://my-app-bucket
   └── awslocal dynamodb create-table ...

3. Run application in local profile
   └── SPRING_PROFILES_ACTIVE=local ./mvnw spring-boot:run

4. Develop & test
   └── Make changes → Test → Repeat

5. Run integration tests against LocalStack
   └── ./mvnw test -Dtest=IntegrationTest

6. Tear down
   └── docker-compose down
```

This workflow keeps your dev cycle fast, cost-free, and isolated from any shared environments.

---

[← Installation & Setup](./02-installation.md) | [Next: Usage Examples →](./04-usage-examples.md)
