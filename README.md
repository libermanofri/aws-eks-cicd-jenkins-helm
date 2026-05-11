# AWS EKS CI/CD Pipeline with Jenkins and Helm

A DevOps project demonstrating the migration of a containerized application from a local Docker-based environment to a cloud-native deployment on **Amazon EKS** using **Jenkins**, **Docker**, **Amazon ECR**, **Helm**, and **Kubernetes**.

The project includes a Python Flask Telegram bot for image processing, a static Nginx website, Docker Compose configuration for local development, a Jenkins CI/CD pipeline, Kubernetes RBAC files, and a Helm chart for deploying the application to EKS.

---

## Table of Contents

- [Overview](#overview)
- [Project Goals](#project-goals)
- [Architecture Overview](#architecture-overview)
- [Tools and Technologies](#tools-and-technologies)
- [Repository Structure](#repository-structure)
- [Application Components](#application-components)
- [CI/CD Pipeline Flow](#cicd-pipeline-flow)
- [Secrets Management](#secrets-management)
- [Helm and Kubernetes Deployment](#helm-and-kubernetes-deployment)
- [Prerequisites](#prerequisites)
- [Local Development Setup](#local-development-setup)
- [EKS Deployment Flow](#eks-deployment-flow)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Security Notes](#security-notes)
- [What I Learned](#what-i-learned)
- [Challenges Faced](#challenges-faced)
- [Possible Improvements and Future Additions](#possible-improvements-and-future-additions)
- [Conclusion](#conclusion)

---

## Overview

The purpose of this project is to demonstrate an end-to-end DevOps workflow for deploying a containerized workload to AWS.

The project starts with an application that can run locally using Docker Compose and continues into a full CI/CD process where Jenkins builds the application image, pushes it to Amazon ECR, packages the Helm chart, and deploys the workload into an EKS namespace.

The main application is a Python Flask-based Telegram bot. The bot receives images from users, applies image-processing filters, and returns the processed image through Telegram.

---

## Project Goals

1. Containerize a Python Flask Telegram bot using Docker.
2. Run the application locally using Docker Compose.
3. Build a CI/CD pipeline using Jenkins.
4. Push application images to Amazon ECR.
5. Deploy the application to Amazon EKS using Helm.
6. Use Kubernetes Secrets to inject sensitive runtime configuration.
7. Demonstrate Kubernetes deployment concepts such as replicas, rolling updates, probes, and image pull configuration.
8. Prepare the project for future improvements such as AWS Secrets Manager integration, monitoring, security scanning, and Infrastructure as Code.

---

## Architecture Overview

```text
Developer
   |
   | Push code to GitHub
   v
GitHub Repository
   |
   | Jenkins Pipeline
   v
Jenkins Agent
   |
   | Build + Test
   v
Docker Image
   |
   | Push image
   v
Amazon ECR
   |
   | Helm upgrade/install
   v
Amazon EKS Cluster
   |
   | Kubernetes Deployment
   v
Python Flask Telegram Bot
```

Local development flow:

```text
Docker Compose
   |
   | Builds local containers
   v
Python Flask App + Nginx Static Website
```

Runtime secret flow:

```text
Kubernetes Secret: telegram-token-secret
   |
   | Injected as environment variable
   v
TELEGRAM_TOKEN inside the application container
   |
   | Read by Flask application
   v
Telegram webhook route and bot logic
```

---

## Tools and Technologies

| Category | Technology |
|---|---|
| Cloud Provider | AWS |
| Kubernetes Platform | Amazon EKS |
| Container Registry | Amazon ECR |
| CI/CD | Jenkins |
| Containerization | Docker |
| Local Multi-Container Setup | Docker Compose |
| Deployment Tool | Helm |
| Orchestration | Kubernetes |
| Application Language | Python |
| Web Framework | Flask |
| Bot Integration | Telegram Bot API |
| Web Server | Nginx |
| Testing | Pytest |
| Secrets | Kubernetes Secret |
| Future Secret Option | AWS Secrets Manager / Secrets Store CSI Driver |

---

## Repository Structure

```text
AWS-Project/
│
├── README.md
├── build.Jenkinsfile
├── compose.yaml
│
├── jenkins-server-k8s/
│   ├── role.yaml
│   ├── rolebinding.yaml
│   └── serviceaccount.yaml
│
├── my-python-app-chart/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── _helpers.tpl
│       └── deployment.yaml
│
├── nginx/
│   ├── Dockerfile
│   ├── index.html
│   └── nginx.conf
│
└── polybot/
    ├── Dockerfile
    ├── app.py
    ├── bot.py
    ├── img_proc.py
    ├── requirements.txt
    ├── app-deployment.yaml
    ├── secret.yaml
    ├── photos/
    └── test/
```

---

## Application Components

### 1. Python Flask Telegram Bot

The main application is located in the `polybot/` directory.

The Flask application exposes routes similar to:

```text
GET /
POST /<TELEGRAM_TOKEN>/
```

The Telegram token is not supposed to be hardcoded in the application code.  
Instead, the app reads it from the `TELEGRAM_TOKEN` environment variable.

Main files:

| File | Description |
|---|---|
| `app.py` | Flask application entry point and Telegram webhook handler |
| `bot.py` | Telegram bot logic and message processing |
| `img_proc.py` | Image-processing functions |
| `requirements.txt` | Python dependencies |
| `Dockerfile` | Docker image definition for the Flask application |
| `test/` | Unit tests executed by Pytest |

The bot supports image-processing actions such as:

- Blur
- Contour
- Segment
- Rotate
- Salt and Pepper
- Concat
- Median
- Edge Extraction

The user sends an image to the Telegram bot with a caption describing the requested filter. The application downloads the image, processes it, and sends the result back to the user.

---

### 2. Nginx Static Website

The `nginx/` directory contains a simple static website served by Nginx.

| File | Description |
|---|---|
| `Dockerfile` | Builds the Nginx image |
| `index.html` | Static website content |
| `nginx.conf` | Nginx server configuration |

This component is useful for demonstrating a multi-service Docker Compose setup.

---

### 3. Docker Compose

The `compose.yaml` file allows local development and testing.

It defines:

| Service | Description |
|---|---|
| `app` | Python Flask Telegram bot |
| `web` | Static Nginx website |

Example command:

```bash
docker compose up --build
```

---

## CI/CD Pipeline Flow

The Jenkins pipeline is defined in:

```text
build.Jenkinsfile
```

The current pipeline performs the following stages:

### 1. Checkout and Extract Git Commit Hash

Jenkins checks out the source code from GitHub and extracts the short Git commit hash.

The commit hash is used as part of the image tag strategy, helping connect a Docker image to the Git version that created it.

---

### 2. Install Python Requirements

The pipeline creates a Python virtual environment and installs the Python packages required for testing and running the application.

---

### 3. Run Unit Tests

The pipeline runs Pytest against the test directory:

```bash
python -m pytest --junitxml results.xml polybot/test
```

The `results.xml` file can be used by Jenkins to display test results.

---

### 4. Login to Amazon ECR

The Jenkins agent authenticates Docker to Amazon ECR using the AWS CLI:

```bash
aws ecr get-login-password --region <AWS_REGION> | docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com
```

The pipeline assumes that the Jenkins agent already has valid AWS permissions, for example through an instance profile, configured AWS credentials, or another secure Jenkins credential mechanism.

---

### 5. Build, Tag, and Push Docker Image

The pipeline builds the Docker image from the `polybot/` directory.

General flow:

```text
cd polybot
docker build
docker tag
docker push to Amazon ECR
```

The image is pushed to an ECR repository and later used by the Helm deployment.

---

### 6. Package Helm Chart

The pipeline updates the Helm chart version and packages the chart:

```bash
helm package ./my-python-app-chart
```

This creates a packaged chart archive that can be installed or upgraded in Kubernetes.

---

### 7. Deploy to EKS with Helm

The pipeline deploys the application using Helm:

```bash
helm upgrade --install my-python-app ./my-python-app-<CHART_VERSION>.tgz   --set image.tag=<IMAGE_TAG>   --atomic --wait   --namespace ofri-test
```

The use of:

```bash
--atomic --wait
```

means Helm waits for the deployment to become healthy. If the deployment fails, Helm automatically rolls back the release.

---

### 8. Trigger Freestyle Job

At the end of the pipeline, Jenkins triggers an additional freestyle job.

In this project, the job is used as a follow-up notification step, for example SNS-related notification handling.

---

## Secrets Management

The application requires a Telegram bot token.

### Current Secret Flow

The application consumes the token through a Kubernetes Secret.

The Helm values define the secret name:

```yaml
env:
  TELEGRAM_TOKEN_SECRET: telegram-token-secret
```

The Kubernetes Deployment template injects the token into the container as an environment variable:

```yaml
env:
  - name: TELEGRAM_TOKEN
    valueFrom:
      secretKeyRef:
        name: {{ .Values.env.TELEGRAM_TOKEN_SECRET }}
        key: TELEGRAM_TOKEN
```

This means the expected Kubernetes Secret is:

```text
Secret name: telegram-token-secret
Secret key:  TELEGRAM_TOKEN
```

The application then reads:

```python
os.environ.get("TELEGRAM_TOKEN")
```

### Important Note About the Pipeline

The current Jenkins pipeline does **not** create or update the `telegram-token-secret`.

Therefore, the secret must already exist in the target namespace before the Helm deployment runs.

Expected flow:

```text
Create Kubernetes Secret manually or by a separate automation
        |
        v
Secret exists in namespace: ofri-test
        |
        v
Jenkins runs Helm deployment
        |
        v
Deployment references the existing secret
        |
        v
Pod receives TELEGRAM_TOKEN as an environment variable
        |
        v
Flask app starts successfully
```

Example command to create the secret:

```bash
kubectl create secret generic telegram-token-secret   --from-literal=TELEGRAM_TOKEN="<YOUR_TELEGRAM_BOT_TOKEN>"   -n ofri-test
```

Do not commit real token values to GitHub.

---

### AWS Secrets Manager / CSI Driver Preparation

The project also contains configuration related to AWS Secrets Manager / AWS Systems Manager Parameter Store through the Secrets Store CSI Driver.

This shows preparation for a more advanced secret-management flow.

However, the current Helm deployment still consumes the Telegram token from a Kubernetes Secret using `secretKeyRef`.

Therefore, the current state is:

```text
Application runtime secret = Kubernetes Secret
AWS Secrets Manager / CSI configuration = prepared or partially configured
```

The future improvement is not that the token is currently exposed.  
The improvement is to complete the integration so the secret is centrally managed in AWS and automatically made available to the EKS workload.

---

## Helm and Kubernetes Deployment

The Helm chart is located in:

```text
my-python-app-chart/
```

Main files:

| File | Description |
|---|---|
| `Chart.yaml` | Helm chart metadata |
| `values.yaml` | Default configuration values |
| `templates/deployment.yaml` | Kubernetes Deployment template |

The deployment template includes:

- Deployment name from Helm values
- Namespace from Helm values
- Replica count
- Rolling update strategy
- Container image repository and tag
- Image pull policy
- Container port `8443`
- `TELEGRAM_TOKEN` environment variable from Kubernetes Secret
- Secret volume mount

The chart values include configuration for:

- Application name
- Namespace
- Replica count
- Image repository
- Image tag
- Resource requests and limits
- Liveness probe timing
- Readiness probe timing
- HPA values
- Ingress host value
- Image pull secret
- SecretProviderClass-related values

---

## Prerequisites

### Local Tools

- Git
- Docker
- Docker Compose
- Python 3
- Pytest
- AWS CLI
- kubectl
- Helm

### AWS Requirements

- AWS account
- Amazon ECR repository
- Amazon EKS cluster
- IAM permissions for ECR and EKS
- kubeconfig configured for the EKS cluster

### Jenkins Requirements

- Jenkins pipeline job
- Jenkins agent with:
  - Docker
  - AWS CLI
  - kubectl
  - Helm
  - Python 3
- AWS permissions available to the Jenkins agent
- Access from Jenkins to the EKS cluster
- ECR repository access
- Existing Kubernetes Secret for the Telegram token

---

## Local Development Setup

### 1. Clone the Repository

```bash
git clone https://github.com/libermanofri/aws-eks-cicd-jenkins-helm.git
cd aws-eks-cicd-jenkins-helm
```

### 2. Configure Telegram Token Locally

Linux/macOS:

```bash
export TELEGRAM_TOKEN="<YOUR_TELEGRAM_BOT_TOKEN>"
```

Windows PowerShell:

```powershell
$env:TELEGRAM_TOKEN="<YOUR_TELEGRAM_BOT_TOKEN>"
```

### 3. Install Python Dependencies

```bash
cd polybot
pip install -r requirements.txt
```

### 4. Run Tests Locally

```bash
python -m pytest test
```

### 5. Run with Docker Compose

From the project root:

```bash
docker compose up --build
```

---

## EKS Deployment Flow

### 1. Configure kubeconfig

```bash
aws eks update-kubeconfig   --region <AWS_REGION>   --name <EKS_CLUSTER_NAME>
```

### 2. Create Namespace

```bash
kubectl create namespace ofri-test
```

### 3. Create the Telegram Token Secret

```bash
kubectl create secret generic telegram-token-secret   --from-literal=TELEGRAM_TOKEN="<YOUR_TELEGRAM_BOT_TOKEN>"   -n ofri-test
```

### 4. Login to Amazon ECR

```bash
aws ecr get-login-password --region <AWS_REGION> | docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com
```

### 5. Build and Push the Application Image

```bash
cd polybot

docker build -t app-image:latest .

docker tag app-image:latest   <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/<ECR_REPOSITORY>:<IMAGE_TAG>

docker push   <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/<ECR_REPOSITORY>:<IMAGE_TAG>
```

### 6. Deploy with Helm

```bash
helm upgrade --install my-python-app ./my-python-app-chart   --namespace ofri-test   --set image.repository=<AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/<ECR_REPOSITORY>   --set image.tag=<IMAGE_TAG>
```

### 7. Verify Deployment

```bash
kubectl get pods -n ofri-test
kubectl get deployments -n ofri-test
kubectl describe pod <POD_NAME> -n ofri-test
kubectl logs <POD_NAME> -n ofri-test
```

---

## Testing

The project uses Pytest.

Run tests manually:

```bash
python -m pytest --junitxml=results.xml polybot/test
```

In Jenkins, the tests are executed as part of the CI/CD pipeline before the Docker image is pushed to ECR.

This ensures that only tested code continues to the image build and deployment stages.

---

## Troubleshooting

### 1. Flask App Fails Because Token Is Missing

Possible error:

```text
No TELEGRAM_TOKEN provided
```

Check that the Kubernetes Secret exists:

```bash
kubectl get secret telegram-token-secret -n ofri-test
```

Check that the pod is using the correct secret name:

```bash
kubectl describe pod <POD_NAME> -n ofri-test
```

---

### 2. ImagePullBackOff

Possible causes:

- Image tag does not exist in ECR.
- EKS nodes do not have permission to pull from ECR.
- Wrong image repository URL.
- Wrong region.
- Missing or incorrect image pull secret.

Check pod events:

```bash
kubectl describe pod <POD_NAME> -n ofri-test
```

---

### 3. Helm Deployment Fails

Check Helm release status:

```bash
helm list -n ofri-test
helm status my-python-app -n ofri-test
```

Check Kubernetes events:

```bash
kubectl get events -n ofri-test --sort-by=.metadata.creationTimestamp
```

---

### 4. Jenkins Cannot Push to ECR

Check:

- AWS CLI is installed on the Jenkins agent.
- Jenkins agent has valid AWS permissions.
- ECR repository exists.
- Region is correct.
- Docker is available on the Jenkins agent.

---

### 5. Jenkins Cannot Deploy to EKS

Check:

- `kubectl` is installed.
- `helm` is installed.
- kubeconfig exists and points to the correct cluster.
- Jenkins agent has network access to the EKS endpoint.
- IAM permissions allow access to the cluster.

---

### 6. Telegram Webhook Does Not Work

Check:

- The Telegram token is valid.
- The application is reachable from the internet.
- The webhook URL is configured correctly.
- The route matches the token-based endpoint.
- The pod logs do not show Flask or Telegram API errors.

---

## Security Notes

- Do not commit real Telegram tokens to GitHub.
- Do not commit AWS access keys or secret keys.
- Use IAM roles or secured Jenkins credentials for AWS access.
- Keep Kubernetes Secrets out of source control.
- Use least-privilege IAM permissions for Jenkins, ECR, and EKS access.
- Consider enabling Kubernetes secret encryption at rest.
- Prefer temporary credentials and IAM roles instead of long-lived static credentials.
- Avoid exposing account-specific values in public documentation when placeholders are enough.

---

## What I Learned

1. How to containerize a Python Flask application.
2. How to build and push Docker images to Amazon ECR.
3. How to deploy workloads to Amazon EKS.
4. How to use Helm for Kubernetes application deployment.
5. How Jenkins can automate build, test, package, and deployment stages.
6. How Kubernetes Secrets inject sensitive values into pods.
7. How image tags and Helm chart versions can be connected to Jenkins build numbers.
8. How to troubleshoot Kubernetes deployment failures.
9. How to prepare a project for more advanced AWS-native integrations.

---

## Challenges Faced

### 1. Managing Runtime Secrets

The application needs a Telegram token at runtime.  
The challenge was to avoid hardcoding the token in the application or Docker image.

The current solution uses a Kubernetes Secret that is injected into the pod as an environment variable.

---

### 2. Connecting Jenkins to AWS and EKS

The Jenkins agent needs access to:

- Amazon ECR for pushing images
- Amazon EKS for deploying workloads
- kubeconfig for cluster communication
- Docker for image build and push

This requires the Jenkins environment to be configured correctly before the pipeline can run successfully.

---

### 3. Helm Chart Versioning

The pipeline packages the Helm chart dynamically and updates the chart version according to the Jenkins build number.

This helps create a clear connection between a Jenkins build and the Helm package deployed to Kubernetes.

---

### 4. Kubernetes Deployment Debugging

When deploying to EKS, errors can come from many places:

- Wrong image tag
- Missing secret
- ECR authentication issue
- Kubernetes namespace issue
- Helm values mismatch
- Pod startup failure

The project helped practice checking pod events, logs, Helm status, and deployment configuration.

---

## Possible Improvements and Future Additions

### 1. Complete AWS Secrets Manager Integration

The project already contains configuration that points toward using AWS Secrets Manager or AWS Systems Manager Parameter Store through the Secrets Store CSI Driver.

Currently, the application consumes the Telegram token from a Kubernetes Secret named `telegram-token-secret`.

A meaningful improvement would be to complete the AWS Secrets Manager integration so the secret is managed centrally in AWS and automatically mounted or synced into the EKS workload.

Benefits:

- Centralized secret management in AWS.
- Better auditability.
- Easier secret rotation.
- Reduced need to manually create Kubernetes Secrets.
- Better alignment with AWS-native security practices.

A possible future flow:

```text
AWS Secrets Manager
   |
   | Secrets Store CSI Driver
   v
EKS Pod
   |
   | Mounted secret or synced Kubernetes Secret
   v
Application receives TELEGRAM_TOKEN
```

---

### 2. Use IAM Roles for Service Accounts (IRSA)

If the application or the CSI driver needs access to AWS resources, IRSA can provide secure, temporary AWS permissions to Kubernetes service accounts.

Instead of storing AWS access keys in Jenkins, Kubernetes, or application configuration, the pod can assume an IAM role through the EKS OIDC provider.

Benefits:

- No long-term AWS keys inside pods.
- Fine-grained permissions per Kubernetes service account.
- Better security and easier permission management.
- Recommended approach for AWS-integrated workloads on EKS.

Example use cases:

- Allowing the Secrets Store CSI Driver to read a specific secret from AWS Secrets Manager.
- Allowing an application pod to access a specific AWS service without static credentials.

---

### 3. Add a Kubernetes Service and Ingress Template to the Helm Chart

The current Helm chart focuses mainly on the Deployment.

A useful improvement would be to add Helm templates for:

- Kubernetes Service
- Ingress
- Optional TLS configuration

This would make the chart more complete and allow the application to be exposed in a controlled and repeatable way.

Benefits:

- Cleaner external access configuration.
- Easier webhook setup for Telegram.
- More production-like Kubernetes architecture.
- Less manual configuration after deployment.

---

### 4. Implement Horizontal Pod Autoscaler Template

The `values.yaml` file already includes HPA-related values such as minimum replicas, maximum replicas, and target CPU utilization.

A future improvement would be to add an actual HPA template to the Helm chart.

Benefits:

- Automatic scaling based on workload.
- Better resource efficiency.
- Improved availability during traffic spikes.
- Stronger demonstration of Kubernetes production concepts.

---

### 5. Add Infrastructure as Code for AWS Resources

The project currently focuses on application deployment, but AWS resources such as EKS, ECR, IAM roles, and networking are assumed to already exist.

A major improvement would be to provision these resources using Terraform.

Resources that can be managed with Terraform:

- EKS cluster
- Node groups
- ECR repositories
- IAM roles and policies
- OIDC provider
- Kubernetes namespaces
- AWS Secrets Manager secrets
- Security groups and networking components

Benefits:

- Repeatable environment creation.
- Better documentation of the infrastructure.
- Easier cleanup and recreation.
- Stronger DevOps portfolio value.

---

### 6. Add Security Scanning to the CI/CD Pipeline

The current pipeline runs unit tests before building and deploying the image.

A meaningful improvement would be to add security scanning before pushing the image to ECR or before deploying to EKS.

Possible scanning areas:

- Python dependencies
- Docker image vulnerabilities
- Kubernetes manifest misconfigurations
- Secrets accidentally committed to the repository

Possible tools:

- Trivy
- Snyk
- Checkov
- kube-score
- kube-linter

Benefits:

- Detect vulnerable dependencies earlier.
- Prevent insecure images from being deployed.
- Improve DevSecOps maturity.
- Show security awareness in the CI/CD process.

---

### 7. Add Monitoring and Logging

A production-grade EKS workload should include better visibility into application behavior.

Future monitoring improvements can include:

- Prometheus for metrics
- Grafana dashboards
- CloudWatch Container Insights
- Centralized application logs
- Alerts for failed pods, high CPU, or deployment failures

Benefits:

- Easier troubleshooting.
- Better operational visibility.
- Faster incident response.
- More realistic production environment.

---

### 8. Parameterize Environment-Specific Values

Some values, such as region, namespace, image repository, and chart settings, should ideally be controlled through Jenkins parameters, Helm values files, or environment-specific configuration.

A future improvement would be to separate values by environment:

```text
values-dev.yaml
values-staging.yaml
values-prod.yaml
```

Benefits:

- Cleaner promotion between environments.
- Less hardcoding.
- Easier reuse of the same chart.
- Reduced risk of deploying to the wrong namespace or cluster.

---

### 9. Add Automated Rollback and Release Documentation

The pipeline already uses Helm with `--atomic --wait`, which helps rollback failed releases.

A useful addition would be to document rollback commands and add a manual rollback process.

Example commands:

```bash
helm history my-python-app -n ofri-test
helm rollback my-python-app <REVISION_NUMBER> -n ofri-test
```

Benefits:

- Clear recovery process.
- Faster response to failed releases.
- Better operational readiness.

---

## Conclusion

This project demonstrates a practical DevOps workflow for migrating and deploying a containerized application to AWS EKS.

It combines Docker, Jenkins, Amazon ECR, Helm, Kubernetes, and EKS into a complete CI/CD process. The project also demonstrates important real-world DevOps concepts such as image versioning, automated testing, Kubernetes deployment, runtime secret injection, and cloud-based container orchestration.

With future improvements such as AWS Secrets Manager integration, IRSA, Terraform-based infrastructure, monitoring, and security scanning, this project can evolve into a stronger production-like cloud deployment solution.

---

## Author

Ofri Liberman

- GitHub: https://github.com/libermanofri
- LinkedIn: https://linkedin.com/in/ofriliberman