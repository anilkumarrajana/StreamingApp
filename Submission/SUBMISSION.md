# Orchestrated Streaming App

This project demonstrates the containerization, continuous integration, orchestration, scaling, monitoring, logging, and notification workflow for a MERN streaming application. The application consists of a React frontend, four Node.js backend services (Auth, Streaming, Admin, and Chat), and MongoDB.

The source application is included in `StreamingApp/.` Infrastructure-as-code resources are included in `helm/streamingapp/` and IAM policies in `iam/`.

# Architecture

![alt text](architecture.png)

The developer pushes source code to GitHub. Jenkins runs on an EC2 instance, builds the Docker images, pushes versioned images to ECR, and sends build notifications through SNS. EKS pulls the images from ECR. An NGINX Ingress load balancer exposes the frontend and API routes. MongoDB stores application data on EBS, while the streaming service uses an IRSA IAM role to access the private S3 bucket. Container Insights sends EKS logs and metrics to CloudWatch.

## 1. Repository Setup

- Fork `UnpredictablePrashant/SampleMERNwithMicroservices`
- Clone your fork:
  - `git clone https://github.com/anilkumarrajana/StreamingApp.git`
  - `cd Streamingapp`
- Sync with upstream:
  - `git remote add upstream https://github.com/UnpredictablePrashant/SampleMERNwithMicroservices.git`
  - `git fetch upstream`
  - `git merge upstream/main`
  - `git push origin main`

## 2.Containerize the Application

- Inspect the source project and Docker configuration
  - `docker compose config --services`
  ![alt text](docker_compose_config.png)
- Verify Docker Desktop
  - `docker info`
  - `docker compose version`
  ![alt text](docker_desktop_check.png)
- Start the MERN Application Locally
  - `docker compose up -d --build`
    ![alt text](docker_compose_up.png)
  - `docker compose ps`
    ![alt text](composeps.png)
  - `docker compose images`
    ![alt text](composeimages.png)
- Check container logs when required

    ```bash
    docker compose logs --tail=100 frontend
    docker compose logs --tail=100 auth
    docker compose logs --tail=100 streaming
    docker compose logs --tail=100 admin
    docker compose logs --tail=100 chat
    ```

- Validate Local Frontend
  - Open `http://localhost:3000` and confirm that the home page loads.
    ![alt text](frontend.png)

## 3.AWS CLI & ECR Setup

- Verify AWS CLI installation
  - `aws --version`
- Configure the named AWS CLI Profile

  ```bash
  aws configure --profile streamingapp
  aws sts get-caller-identity --profile streamingapp
  ```
  
  ![alt text](AWS_profile.png)

- Set reusable local variables
  
  ```powershell
    $env:AWS_PROFILE = "streamingapp"
    $env:AWS_REGION  = "ap-south-1"
    $AWS_ACCOUNT_ID = aws sts get-caller-identity --profile $env:AWS_PROFILE --query Account --output text
    $env:ECR_REGISTRY = "$AWS_ACCOUNT_ID.dkr.ecr.$env:AWS_REGION.amazonaws.com"
  ```

- Create private ECR repositories

```Powershell
  $repos = @(
    "streaming-frontend",
    "streaming-auth",
    "streaming-streaming",
    "streaming-admin",
    "streaming-chat"
  )

  foreach ($repo in $repos) {
    aws ecr create-repository `
      --repository-name $repo `
      --image-scanning-configuration scanOnPush=true `
      --region $env:AWS_REGION `
      --profile $env:AWS_PROFILE
  }

```

- Verify ECR Repositories

```bash
  aws ecr describe-repositories --region ap-south-1 --profile streamingapp --query "repositories[].repositoryName" --output table
```

![alt text](ECR_Repos.png)

![alt text](ECR_Repos_CLI.png)

- Log Docker in to ECR

```bash
aws ecr get-login-password --region "$AWS_REGION" --profile "$AWS_PROFILE" | \
docker login --username AWS --password-stdin "$ECR_REGISTRY"
```

![alt text](ECR_docker_login.png)

- Build the frontend image for AMD64

    ```bash
      docker buildx build --platform linux/amd64 `
      -t "$env:ECR_REGISTRY/streaming-frontend:$env:IMAGE_TAG" `
      --push `
      .\frontend
    ```
![alt text](FE_image.png)

- Build Image for Authentication Service in backend

  ```bash
    docker buildx build --platform linux/amd64 `
      -t "$env:ECR_REGISTRY/streaming-auth:$env:IMAGE_TAG" `
      --push `
      .\backend\authService
  ```
  ![alt text](AS_image.png)
- Build Image for Streaming Service in backend

```bash
docker buildx build --platform linux/amd64 `
  -t "$env:ECR_REGISTRY/streaming-streaming:$env:IMAGE_TAG" `
  --push `
  .\backend\streamingService
```
![alt text](SS_image.png)

- Build Image for Admin Service in Backend

```bash
docker buildx build --platform linux/amd64 `
  -t "$env:ECR_REGISTRY/streaming-admin:$env:IMAGE_TAG" `
  --push `
  .\backend\adminService
```
![alt text](SA_image.png)

- Building Image for Chat Service in Backend
```bash
docker buildx build --platform linux/amd64 `
  -t "$env:ECR_REGISTRY/streaming-chat:$env:IMAGE_TAG" `
  --push `
  .\backend\chatService
```
![alt text](CS_image.png)

- Verify the Image architecture and ECR Tags
```bash
$env:AWS_REGION = "ap-south-1"
$env:AWS_PROFILE = "streamingapp"

$repos = @(
  "streaming-frontend",
  "streaming-auth",
  "streaming-streaming",
  "streaming-admin",
  "streaming-chat"
)

foreach ($repo in $repos) {
  Write-Host "`n$repo"
  aws ecr describe-images `
    --repository-name $repo `
    --region $env:AWS_REGION `
    --profile $env:AWS_PROFILE `
    --query "imageDetails[].imageTags" `
    --output table
}
```
![alt text](ECR_Tags_Validation.png)