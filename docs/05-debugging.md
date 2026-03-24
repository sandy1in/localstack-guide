# 05 · Debugging LocalStack

[← Usage Examples](./04-usage-examples.md) | [Next: Further Resources →](./06-resources.md)

---

## Table of Contents

- [Health Check](#health-check)
- [Viewing Logs](#viewing-logs)
- [LocalStack Web Dashboard](#localstack-web-dashboard)
- [Common Errors and Fixes](#common-errors-and-fixes)
- [Debugging Lambda Functions](#debugging-lambda-functions)
- [Inspecting Service State](#inspecting-service-state)
- [Resetting LocalStack](#resetting-localstack)
- [Debug Tips for CI/CD Pipelines](#debug-tips-for-cicd-pipelines)

---

## Health Check

The first thing to check when something isn't working is LocalStack's health endpoint:

```bash
curl http://localhost:4566/_localstack/health | python3 -m json.tool
```

A healthy response looks like:

```json
{
  "services": {
    "s3": "running",
    "dynamodb": "running",
    "sqs": "running",
    "sns": "running",
    "lambda": "running",
    "iam": "running",
    "kms": "running"
  },
  "version": "3.x.x"
}
```

If a service shows `"error"` or is missing, it may not have started correctly. Restart LocalStack and check the container logs.

---

## Viewing Logs

### Docker Compose logs

```bash
# Stream logs from LocalStack container
docker-compose logs -f localstack

# View last 100 lines
docker-compose logs --tail=100 localstack
```

### Direct Docker logs

```bash
# Find container name
docker ps

# Stream logs
docker logs -f localstack_dev

# Last N lines
docker logs --tail=200 localstack_dev
```

### Enable debug mode

Set `DEBUG=1` in your Docker Compose environment to get verbose output including all incoming API requests and responses:

```yaml
environment:
  - DEBUG=1
```

This is invaluable when you want to see exactly what API calls your application is making and whether LocalStack is receiving them correctly.

---

## LocalStack Web Dashboard

LocalStack Pro includes a web dashboard at `http://localhost:8080` that gives you a visual interface to:
- Browse S3 buckets and objects
- View DynamoDB tables and items
- Monitor SQS queues and messages
- Inspect Lambda functions and logs
- Track all API calls in real time

For the Community edition, the **LocalStack Desktop** app (available at [app.localstack.cloud](https://app.localstack.cloud)) provides similar functionality for free when connected to a running local instance.

---

## Common Errors and Fixes

### `Could not connect to the endpoint URL: http://localhost:4566`

**Cause**: LocalStack is not running.

**Fix**:
```bash
docker ps | grep localstack
docker-compose up -d
```

---

### `An error occurred (NoSuchBucket)`

**Cause**: The bucket you're referencing doesn't exist.

**Fix**: Create it first, or check for typos in the bucket name:
```bash
awslocal s3 mb s3://my-bucket
awslocal s3 ls   # verify it exists
```

---

### `S3 SignatureDoesNotMatch` or credential errors

**Cause**: AWS SDK is not configured to skip credential validation when pointing to LocalStack.

**Fix**: Use dummy credentials and disable validation:
```properties
# Spring Boot
cloud.aws.credentials.access-key=test
cloud.aws.credentials.secret-key=test
```

```java
// Java SDK v1
.withCredentials(new AWSStaticCredentialsProvider(
    new BasicAWSCredentials("test", "test")))
```

---

### S3 bucket URL resolving to `my-bucket.localhost:4566` instead of `localhost:4566/my-bucket`

**Cause**: Virtual-hosted-style S3 URLs are not compatible with LocalStack.

**Fix**: Enable path-style access:
```java
.withPathStyleAccessEnabled(true)
```
```python
config=Config(s3={'addressing_style': 'path'})
```
```javascript
s3ForcePathStyle: true
```

---

### Lambda function times out or doesn't execute

**Cause**: Docker-in-Docker issues (Lambda runs in its own container inside LocalStack).

**Fix**: Ensure the Docker socket is mounted:
```yaml
volumes:
  - "/var/run/docker.sock:/var/run/docker.sock"
```

Also confirm the Lambda runtime is supported:
```bash
awslocal lambda list-functions
awslocal lambda get-function --function-name my-function
```

---

### `ResourceNotFoundException` for DynamoDB

**Cause**: Table doesn't exist, or wrong region.

**Fix**:
```bash
awslocal dynamodb list-tables
awslocal dynamodb describe-table --table-name MyTable
```

---

### Port `4566` already in use

**Cause**: Another process (or a previous LocalStack container) is using the port.

**Fix**:
```bash
# Find what's using the port
sudo lsof -i :4566

# Kill the process or stop the conflicting container
docker-compose down
```

---

### `docker: permission denied` in WSL2

**Cause**: Current user is not in the `docker` group.

**Fix**:
```bash
sudo usermod -aG docker $USER
exec $SHELL   # or log out and back in
```

---

## Debugging Lambda Functions

Lambda debugging in LocalStack is more involved because functions run in isolated containers.

### View Lambda logs

```bash
# Get the log group name
awslocal logs describe-log-groups

# Get log streams
awslocal logs describe-log-streams \
  --log-group-name /aws/lambda/my-function

# Read log events
awslocal logs get-log-events \
  --log-group-name /aws/lambda/my-function \
  --log-stream-name <stream-name>
```

### Use console output for quick debugging

Add `console.log` / `print` statements to your Lambda function. These will appear in the LocalStack container logs when the function is invoked:

```bash
docker logs -f localstack_dev | grep "my-function"
```

### Test Lambda locally before deploying

Use the AWS SAM CLI with LocalStack:

```bash
sam local invoke MyFunction --event event.json \
  --env-vars env.json
```

---

## Inspecting Service State

### List all S3 objects across all buckets

```bash
for bucket in $(awslocal s3 ls | awk '{print $3}'); do
  echo "=== $bucket ===";
  awslocal s3 ls s3://$bucket --recursive;
done
```

### Check SQS queue depth

```bash
awslocal sqs get-queue-attributes \
  --queue-url http://localhost:4566/000000000000/my-queue \
  --attribute-names ApproximateNumberOfMessages
```

### Check DynamoDB item count

```bash
awslocal dynamodb describe-table --table-name Users \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('Items:', d['Table']['ItemCount'])"
```

---

## Resetting LocalStack

### Soft reset (clear all resources, keep container running)

```bash
curl -X POST http://localhost:4566/_localstack/state/reset
```

### Hard reset (remove container and data)

```bash
docker-compose down -v        # removes volumes too
docker-compose up -d          # fresh start
```

### Selective cleanup

```bash
# Delete all S3 buckets
for bucket in $(awslocal s3 ls | awk '{print $3}'); do
  awslocal s3 rb s3://$bucket --force
done

# Delete all DynamoDB tables
for table in $(awslocal dynamodb list-tables | python3 -c "import sys,json; [print(t) for t in json.load(sys.stdin)['TableNames']]"); do
  awslocal dynamodb delete-table --table-name $table
done
```

---

## Debug Tips for CI/CD Pipelines

When using LocalStack in GitHub Actions, GitLab CI, or Jenkins:

### GitHub Actions example

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      localstack:
        image: localstack/localstack:latest
        ports:
          - 4566:4566
        env:
          SERVICES: s3,dynamodb,sqs
          DEBUG: 1
        options: >-
          --health-cmd "curl -f http://localhost:4566/_localstack/health"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3
      - name: Wait for LocalStack
        run: |
          until curl -s http://localhost:4566/_localstack/health | grep '"s3": "running"'; do
            echo "Waiting for LocalStack..."; sleep 2;
          done
      - name: Run Tests
        run: mvn test
```

### Tips

- Always add a **readiness check** before running tests — LocalStack takes a few seconds to fully start.
- Use `--health-cmd` in Docker service definitions to gate tests on LocalStack being ready.
- Set `DEBUG=1` in CI to get full request/response logs when a test fails.
- Cache the LocalStack Docker image in CI to speed up pipeline runs.
- Use Testcontainers in Java/Python to manage the LocalStack lifecycle programmatically within tests.

---

[← Usage Examples](./04-usage-examples.md) | [Next: Further Resources →](./06-resources.md)
