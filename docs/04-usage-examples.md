# 04 · Usage Examples

[← Core Concepts](./03-core-concepts.md) | [Next: Debugging →](./05-debugging.md)

---

## Table of Contents

- [AWS CLI Examples](#aws-cli-examples)
  - [S3](#s3)
  - [SQS](#sqs)
  - [DynamoDB](#dynamodb)
  - [SNS](#sns)
  - [Lambda](#lambda)
  - [IAM](#iam)
  - [KMS](#kms)
- [Spring Boot Integration](#spring-boot-integration)
- [Terraform Example](#terraform-example)
- [Shell Script Wrapper](#shell-script-wrapper)
- [Python (boto3) Examples](#python-boto3-examples)
- [Testing with LocalStack (JUnit)](#testing-with-localstack-junit)

---

## AWS CLI Examples

> All examples use `awslocal` (the LocalStack wrapper). If you haven't installed it, replace `awslocal` with `aws --endpoint-url=http://localhost:4566`.

### S3

```bash
# Create a bucket
awslocal s3 mb s3://my-local-bucket

# Upload a file
awslocal s3 cp ./test.txt s3://my-local-bucket/

# List objects in a bucket
awslocal s3 ls s3://my-local-bucket/

# Download a file
awslocal s3 cp s3://my-local-bucket/test.txt ./downloaded.txt

# Delete an object
awslocal s3 rm s3://my-local-bucket/test.txt

# Delete a bucket (must be empty)
awslocal s3 rb s3://my-local-bucket

# Sync a local directory to S3
awslocal s3 sync ./dist s3://my-local-bucket/

# Generate a pre-signed URL (expires in 300 seconds)
awslocal s3 presign s3://my-local-bucket/test.txt --expires-in 300
```

---

### SQS

```bash
# Create a standard queue
awslocal sqs create-queue --queue-name my-queue

# Create a FIFO queue
awslocal sqs create-queue \
  --queue-name my-queue.fifo \
  --attributes FifoQueue=true,ContentBasedDeduplication=true

# List all queues
awslocal sqs list-queues

# Send a message
awslocal sqs send-message \
  --queue-url http://localhost:4566/000000000000/my-queue \
  --message-body "Hello from LocalStack"

# Receive messages
awslocal sqs receive-message \
  --queue-url http://localhost:4566/000000000000/my-queue \
  --max-number-of-messages 5

# Delete a message (use ReceiptHandle from receive-message output)
awslocal sqs delete-message \
  --queue-url http://localhost:4566/000000000000/my-queue \
  --receipt-handle <ReceiptHandle>

# Get queue attributes
awslocal sqs get-queue-attributes \
  --queue-url http://localhost:4566/000000000000/my-queue \
  --attribute-names All
```

---

### DynamoDB

```bash
# Create a table
awslocal dynamodb create-table \
  --table-name Users \
  --attribute-definitions \
    AttributeName=UserId,AttributeType=S \
  --key-schema \
    AttributeName=UserId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

# Put an item
awslocal dynamodb put-item \
  --table-name Users \
  --item '{"UserId": {"S": "user-001"}, "Name": {"S": "Alice"}, "Age": {"N": "30"}}'

# Get an item
awslocal dynamodb get-item \
  --table-name Users \
  --key '{"UserId": {"S": "user-001"}}'

# Scan a table (returns all items — use Query for production)
awslocal dynamodb scan --table-name Users

# Query with a filter
awslocal dynamodb query \
  --table-name Users \
  --key-condition-expression "UserId = :uid" \
  --expression-attribute-values '{":uid": {"S": "user-001"}}'

# Delete an item
awslocal dynamodb delete-item \
  --table-name Users \
  --key '{"UserId": {"S": "user-001"}}'

# List all tables
awslocal dynamodb list-tables
```

---

### SNS

```bash
# Create a topic
awslocal sns create-topic --name my-topic

# List topics
awslocal sns list-topics

# Subscribe an SQS queue to the topic
awslocal sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:000000000000:my-topic \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:000000000000:my-queue

# Publish a message to the topic
awslocal sns publish \
  --topic-arn arn:aws:sns:us-east-1:000000000000:my-topic \
  --message "Hello subscribers!"

# List subscriptions
awslocal sns list-subscriptions
```

---

### Lambda

```bash
# Package a simple function (Node.js example)
mkdir lambda-fn && cd lambda-fn
cat > index.js << 'EOF'
exports.handler = async (event) => {
  console.log('Event:', JSON.stringify(event));
  return { statusCode: 200, body: 'Hello from LocalStack Lambda!' };
};
EOF
zip function.zip index.js

# Create the Lambda function
awslocal lambda create-function \
  --function-name my-function \
  --runtime nodejs18.x \
  --handler index.handler \
  --zip-file fileb://function.zip \
  --role arn:aws:iam::000000000000:role/lambda-role

# Invoke the function
awslocal lambda invoke \
  --function-name my-function \
  --payload '{"key": "value"}' \
  output.json

cat output.json

# List functions
awslocal lambda list-functions

# Update function code
zip function.zip index.js
awslocal lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip

# Delete the function
awslocal lambda delete-function --function-name my-function
```

---

### IAM

```bash
# Create a user
awslocal iam create-user --user-name dev-user

# List users
awslocal iam list-users

# Create an access key for the user
awslocal iam create-access-key --user-name dev-user

# Create a role
awslocal iam create-role \
  --role-name lambda-role \
  --assume-role-policy-document file://trust-policy.json

# Attach a managed policy
awslocal iam attach-role-policy \
  --role-name lambda-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

---

### KMS

```bash
# Create a KMS key
awslocal kms create-key --description "my-test-key"

# List keys
awslocal kms list-keys

# Encrypt data
awslocal kms encrypt \
  --key-id <key-id> \
  --plaintext "Hello World" \
  --query CiphertextBlob \
  --output text

# Decrypt data
awslocal kms decrypt \
  --ciphertext-blob <blob> \
  --output text \
  --query Plaintext | base64 --decode
```

---

## Spring Boot Integration

### Maven Dependencies (`pom.xml`)

```xml
<dependency>
  <groupId>io.awspring.cloud</groupId>
  <artifactId>spring-cloud-starter-aws-s3</artifactId>
</dependency>
<dependency>
  <groupId>io.awspring.cloud</groupId>
  <artifactId>spring-cloud-starter-aws-messaging</artifactId>
</dependency>
```

### Application Properties

`application-local.properties`:
```properties
cloud.aws.endpoint=http://localhost:4566
cloud.aws.region.static=us-east-1
cloud.aws.credentials.access-key=test
cloud.aws.credentials.secret-key=test
cloud.aws.stack.auto=false
```

### Conditional S3 Bean

```java
@Configuration
public class AwsConfig {

    @Value("${cloud.aws.endpoint:}")
    private String endpoint;

    @Value("${cloud.aws.region.static:us-east-1}")
    private String region;

    @Bean
    @Profile("local")
    public AmazonS3 localS3() {
        return AmazonS3ClientBuilder.standard()
            .withEndpointConfiguration(
                new AwsClientBuilder.EndpointConfiguration(endpoint, region))
            .withPathStyleAccessEnabled(true)  // Required for LocalStack
            .withCredentials(new AWSStaticCredentialsProvider(
                new BasicAWSCredentials("test", "test")))
            .build();
    }

    @Bean
    @Profile("prod")
    public AmazonS3 prodS3() {
        return AmazonS3ClientBuilder.standard()
            .withRegion(region)
            .build(); // Uses IAM role or environment credentials
    }
}
```

### S3 Service Example

```java
@Service
public class S3Service {

    @Autowired
    private AmazonS3 s3;

    private static final String BUCKET = "my-app-bucket";

    public void uploadFile(String key, String content) {
        if (!s3.doesBucketExistV2(BUCKET)) {
            s3.createBucket(BUCKET);
        }
        s3.putObject(BUCKET, key, content);
        System.out.println("Uploaded: " + key);
    }

    public String downloadFile(String key) {
        return s3.getObjectAsString(BUCKET, key);
    }

    public void deleteFile(String key) {
        s3.deleteObject(BUCKET, key);
    }
}
```

### SQS Listener Example

```java
@Component
public class MyQueueListener {

    @SqsListener("my-queue")
    public void handleMessage(String message) {
        System.out.println("Received: " + message);
        // Process the message
    }
}
```

Switch profiles:
```bash
SPRING_PROFILES_ACTIVE=local   mvn spring-boot:run   # LocalStack
SPRING_PROFILES_ACTIVE=prod    mvn spring-boot:run   # Real AWS
```

---

## Terraform Example

### `main.tf`

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
    s3       = "http://localhost:4566"
    dynamodb = "http://localhost:4566"
    sqs      = "http://localhost:4566"
    lambda   = "http://localhost:4566"
    iam      = "http://localhost:4566"
  }
}

resource "aws_s3_bucket" "app_assets" {
  bucket = "app-assets-local"
}

resource "aws_dynamodb_table" "users" {
  name           = "Users"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "UserId"

  attribute {
    name = "UserId"
    type = "S"
  }
}

resource "aws_sqs_queue" "app_queue" {
  name = "app-queue"
}
```

```bash
terraform init
terraform apply -auto-approve
terraform destroy -auto-approve
```

---

## Shell Script Wrapper

A utility script to avoid typing the endpoint URL every time:

`localstack-aws.sh`:

```bash
#!/bin/bash
# LocalStack AWS CLI wrapper
PROFILE="localstack"
ENDPOINT="http://localhost:4566"

# Configure profile once
aws configure set aws_access_key_id test --profile $PROFILE
aws configure set aws_secret_access_key test --profile $PROFILE
aws configure set region us-east-1 --profile $PROFILE

SERVICE=$1
ACTION=$2

case "$SERVICE:$ACTION" in
  s3:create)        awslocal s3 mb s3://my-local-bucket ;;
  s3:list)          awslocal s3 ls ;;
  dynamodb:create)  awslocal dynamodb create-table \
                      --table-name MyTable \
                      --attribute-definitions AttributeName=Id,AttributeType=S \
                      --key-schema AttributeName=Id,KeyType=HASH \
                      --billing-mode PAY_PER_REQUEST ;;
  dynamodb:list)    awslocal dynamodb list-tables ;;
  sqs:create)       awslocal sqs create-queue --queue-name my-queue ;;
  sqs:list)         awslocal sqs list-queues ;;
  sns:create)       awslocal sns create-topic --name my-topic ;;
  sns:list)         awslocal sns list-topics ;;
  lambda:list)      awslocal lambda list-functions ;;
  iam:list)         awslocal iam list-users ;;
  kms:list)         awslocal kms list-keys ;;
  *)  echo "Usage: $0 {s3|dynamodb|sqs|sns|lambda|iam|kms} {create|list}" ;;
esac
```

```bash
chmod +x localstack-aws.sh

./localstack-aws.sh s3 create
./localstack-aws.sh dynamodb list
./localstack-aws.sh sqs create
```

---

## Python (boto3) Examples

```python
import boto3

# LocalStack client factory
def get_client(service):
    return boto3.client(
        service,
        endpoint_url='http://localhost:4566',
        aws_access_key_id='test',
        aws_secret_access_key='test',
        region_name='us-east-1'
    )

# S3 example
s3 = get_client('s3')
s3.create_bucket(Bucket='my-py-bucket')
s3.put_object(Bucket='my-py-bucket', Key='hello.txt', Body=b'Hello LocalStack!')
response = s3.get_object(Bucket='my-py-bucket', Key='hello.txt')
print(response['Body'].read().decode())

# SQS example
sqs = get_client('sqs')
queue = sqs.create_queue(QueueName='my-py-queue')
queue_url = queue['QueueUrl']
sqs.send_message(QueueUrl=queue_url, MessageBody='Hello from Python!')
messages = sqs.receive_message(QueueUrl=queue_url)
print(messages['Messages'][0]['Body'])
```

---

## Testing with LocalStack (JUnit)

### Using Testcontainers

The cleanest way to use LocalStack in automated tests is with the [Testcontainers](https://testcontainers.com/) library, which spins up LocalStack as a container for each test run.

```xml
<!-- pom.xml -->
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>localstack</artifactId>
  <version>1.19.0</version>
  <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
@Testcontainers
class S3ServiceIntegrationTest {

    @Container
    static LocalStackContainer localStack = new LocalStackContainer(
        DockerImageName.parse("localstack/localstack:latest"))
        .withServices(LocalStackContainer.Service.S3);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("cloud.aws.endpoint",
            () -> localStack.getEndpointOverride(LocalStackContainer.Service.S3).toString());
        registry.add("cloud.aws.region.static", localStack::getRegion);
        registry.add("cloud.aws.credentials.access-key", localStack::getAccessKey);
        registry.add("cloud.aws.credentials.secret-key", localStack::getSecretKey);
    }

    @Autowired
    private S3Service s3Service;

    @Test
    void shouldUploadAndDownloadFile() {
        s3Service.uploadFile("test.txt", "Hello Test!");
        String content = s3Service.downloadFile("test.txt");
        assertEquals("Hello Test!", content);
    }
}
```

> Testcontainers automatically starts and stops the LocalStack container for each test class, giving you fully isolated and reproducible integration tests.

---

[← Core Concepts](./03-core-concepts.md) | [Next: Debugging →](./05-debugging.md)
