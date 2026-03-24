# 02 · Installation & Setup

[← Introduction](./01-introduction.md) | [Next: Core Concepts →](./03-core-concepts.md)

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: WSL2 & System Setup](#step-1-wsl2--system-setup)
- [Step 2: Install Docker on WSL2](#step-2-install-docker-on-wsl2)
- [Step 3: Install LocalStack](#step-3-install-localstack)
- [Step 4: Install AWS CLI and awscli-local](#step-4-install-aws-cli-and-awscli-local)
- [Step 5: Run LocalStack with Docker Compose](#step-5-run-localstack-with-docker-compose)
- [Step 6: Configure AWS CLI for LocalStack](#step-6-configure-aws-cli-for-localstack)
- [Step 7: IntelliJ IDEA Setup](#step-7-intellij-idea-setup)
- [Verifying Your Setup](#verifying-your-setup)

---

## Prerequisites

Before you begin, ensure the following are available on your machine:

| Requirement | Notes |
|-------------|-------|
| **Windows with WSL2** | Ubuntu recommended |
| **Docker Desktop or Docker Engine** | Inside WSL2 |
| **AWS CLI v2** | For interacting with LocalStack via terminal |
| **awscli-local** | Wrapper that automatically uses LocalStack endpoint |
| **IntelliJ IDEA** | May 2025 version or later recommended |
| **Java 17+** | For Spring Boot examples |
| **Terraform** | Optional, for IaC examples |

---

## Step 1: WSL2 & System Setup

Open PowerShell as Administrator and verify WSL2 is active:

```bash
# Check WSL version
wsl --list --verbose

# (Optional) Convert an existing distro to WSL2
wsl --set-version Ubuntu 2

# Update packages inside WSL2 (Ubuntu)
sudo apt update && sudo apt upgrade -y
```

> **Tip:** WSL2 is required over WSL1 because Docker Engine uses Linux kernel features only available in WSL2.

---

## Step 2: Install Docker on WSL2

```bash
# Install dependencies
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add the Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Start Docker and grant current user access (avoids needing sudo)
sudo service docker start
sudo usermod -aG docker $USER

# Restart shell to apply group changes
exec $SHELL

# Verify installation
docker --version
docker run hello-world
```

> If `hello-world` runs successfully, Docker is installed and working correctly.

### Configure Docker daemon for WSL2

Edit the Docker daemon config:

```bash
sudo nano /etc/docker/daemon.json
```

Add the following:

```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": { "max-size": "100m" },
  "storage-driver": "overlay2",
  "hosts": ["unix:///var/run/docker.sock"]
}
```

Also update the Docker service file to avoid conflicts:

```bash
sudo nano /usr/lib/systemd/system/docker.service
```

Ensure the `ExecStart` line reads:

```
ExecStart=/usr/bin/dockerd --containerd=/run/containerd/containerd.sock
```

---

## Step 3: Install LocalStack

```bash
# Install pipx (Python tool isolator)
sudo apt install -y pipx

# Install LocalStack CLI with all dependencies
pipx install localstack --include-deps

# Add LocalStack to PATH
echo 'export PATH=$PATH:/root/.local/bin' >> ~/.bashrc
source ~/.bashrc

# Verify installation
localstack --version

# Start LocalStack in detached mode
localstack start -d
```

> `localstack start -d` pulls and starts the LocalStack Docker image in the background.

---

## Step 4: Install AWS CLI and awscli-local

```bash
# Install AWS CLI v2
sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version

# Install awscli-local (wraps every command with --endpoint-url automatically)
pipx install awscli-local

# Optional fallback via apt
sudo apt install -y awscli
```

### What is `awscli-local`?

`awscli-local` is a thin wrapper around the standard `aws` CLI. Instead of typing:

```bash
aws --endpoint-url=http://localhost:4566 s3 ls
```

You can simply type:

```bash
awslocal s3 ls
```

It saves time and reduces errors, especially when running many commands during development.

---

## Step 5: Run LocalStack with Docker Compose

Using Docker Compose is the recommended way to manage LocalStack in a team or project environment. It ensures everyone uses the same configuration.

Create a `docker-compose.yml` in your project root:

```yaml
version: "3.8"

services:
  localstack:
    image: localstack/localstack:latest
    container_name: localstack_dev
    ports:
      - "4566:4566"       # Main LocalStack gateway
      - "4510-4559:4510-4559"  # Optional external service ports
    environment:
      - SERVICES=s3,dynamodb,sqs,sns,iam,kms,elasticache,ec2,lambda,route53,cloudfront,elbv2,eks
      - DEBUG=1
      - DATA_DIR=/tmp/localstack/data
      - AWS_DEFAULT_REGION=us-east-1
      - AWS_ACCESS_KEY_ID=test
      - AWS_SECRET_ACCESS_KEY=test
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "./localstack-data:/tmp/localstack/data"  # Optional: persist state
```

### Manage the containers

```bash
# Start in detached mode
docker-compose up -d

# Check running containers
docker ps

# View LocalStack logs
docker-compose logs -f localstack

# Stop and remove containers
docker-compose down

# Stop and remove containers + volumes (clean slate)
docker-compose down -v
```

> **Pro tip:** Add `docker-compose.yml` to your project repo so any team member can run the same local environment with a single command.

---

## Step 6: Configure AWS CLI for LocalStack

LocalStack accepts any dummy credentials. Configure a dedicated AWS profile for LocalStack:

```bash
aws configure --profile localstack
# AWS Access Key ID:     test
# AWS Secret Access Key: test
# Default region name:   us-east-1
# Default output format: json
```

Test the connection:

```bash
aws --profile localstack --endpoint-url=http://localhost:4566 s3 ls
```

Or using `awscli-local`:

```bash
awslocal s3 ls
```

---

## Step 7: IntelliJ IDEA Setup

### Connect IntelliJ to Docker in WSL2

1. Open **File → Settings → Build, Execution, Deployment → Docker**
2. Click **+** to add a new Docker connection
3. Select **Docker inside WSL2**
4. Choose connection via **Unix socket**: `unix:///var/run/docker.sock`
5. Click **Apply** — IntelliJ will verify the connection

Once connected, you can manage Docker containers, view logs, and start/stop services directly from the **Services** panel inside IntelliJ.

### Run Docker Compose from IntelliJ

Right-click on `docker-compose.yml` in the project tree → **Run** → IntelliJ will spin up the defined services.

---

## Verifying Your Setup

Run this quick checklist to confirm everything is working:

```bash
# 1. LocalStack container is running
docker ps | grep localstack

# 2. LocalStack health check
curl http://localhost:4566/_localstack/health | python3 -m json.tool

# 3. Create a test S3 bucket
awslocal s3 mb s3://test-setup-bucket

# 4. List buckets
awslocal s3 ls

# 5. Create a DynamoDB table
awslocal dynamodb create-table \
  --table-name SetupCheck \
  --attribute-definitions AttributeName=Id,AttributeType=S \
  --key-schema AttributeName=Id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

# 6. List tables
awslocal dynamodb list-tables
```

Expected output for health check:

```json
{
  "services": {
    "s3": "running",
    "dynamodb": "running",
    "sqs": "running",
    ...
  }
}
```

---

[← Introduction](./01-introduction.md) | [Next: Core Concepts →](./03-core-concepts.md)
