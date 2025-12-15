# Learn Jenkins App – CI/CD with Jenkins, Docker & AWS ECS

This repository demonstrates a production-style CI/CD pipeline that builds a React application, packages it into a Docker image, and deploys it to AWS ECS using Jenkins.

---

## Architecture Overview

The pipeline is intentionally split into **CI-only tooling containers** and a **production runtime container**.

GitHub → Jenkins → Docker → Amazon ECR → Amazon ECS

## Repository Structure

├── Dockerfile                     # Production image (used by ECS)

├── Jenkinsfile-aws                 # Main Jenkins CI/CD pipeline

├── aws/

│   └── task-definition-prod.json  # ECS task definition

├── ci/

│   ├── Dockerfile-aws-cli          # Jenkins CI tool image (aws, docker, jq)

│   └── Dockerfile-playwright       # CI test image (optional)

├── src/                            # React application source

├── build/                          # Generated static build output

├── package.json

└── package-lock.json

## Dockerfiles Explained

### 1. Root Dockerfile (Production)

- Builds the **runtime container**
- This image is deployed to ECS
- Contains only what is needed to run the application

```text
Used by: Amazon ECS
Built by: Jenkins

### 2. ci/Dockerfile-aws-cli (CI Tooling)

- Used only inside Jenkins pipeline stages
- Provides:
    - AWS CLI
    - Docker CLI
    - jq

```
Used by: Jenkins only
Never deployed

```

---

## Jenkins Pipeline Stages

### Stage 1 – Build Application

- Runs inside `node:18-alpine`
- Installs dependencies
- Builds the React app

```bash
npm ci
npm run build

```

---

### Stage 2 – Build & Push Docker Image

- Runs inside custom `my-aws-cli` image
- Uses Docker buildx to force `linux/amd64`
- Pushes image to Amazon ECR

```bash
docker buildx build \
  --platform linux/amd64 \
  -t <ECR_REPO>:<BUILD_TAG> \
  --push .

```

---

### Stage 3 – Deploy to ECS

- Registers a new ECS task definition revision
- Updates ECS service
- ECS performs rolling deployment

---

## ECS Deployment Model

- ECS Service runs tasks from the latest task definition revision
- Each deployment updates only the task definition (immutable infra)
- Container logs are forwarded to CloudWatch Logs

---

## Platform Considerations

### ARM vs AMD64

The pipeline explicitly builds `linux/amd64` images to ensure compatibility with ECS runtimes.

This avoids runtime errors such as:

```
exec format error

```

which occur when ARM images are deployed onto x86 infrastructure.

---

## Key Takeaways

- Clear separation of CI tooling and production images
- Immutable Docker images promoted through environments
- ECS-native deployment process using task definitions
- Architecture-aware Docker builds


```
Jenkins → Docker → ECR → ECS
┌───────────────────────────────────────────┐
│            GitHub Repository               │
│                                           │
│  ├── Jenkinsfile-aws                       │
│  ├── Dockerfile          (PROD IMAGE) ✅  │
│  ├── ci/                                   │
│  │    └── Dockerfile-aws-cli (CI TOOLS)   │
│  ├── src/                                 │
│  ├── build/                               │
│  └── aws/task-definition-prod.json        │
└───────────────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────┐
│                 Jenkins                   │
│                                           │
│  Pipeline stages run in containers         │
│                                           │
│  ┌───────────────┐                        │
│  │ Stage 1       │                        │
│  │ Build App     │                        │
│  │               │                        │
│  │ node:18       │  npm ci                │
│  │ (CI only)     │  npm run build          │
│  └───────────────┘                        │
│          │                                │
│          ▼                                │
│  ┌───────────────┐                        │
│  │ Stage 2       │                        │
│  │ Build Image   │                        │
│  │               │                        │
│  │ my-aws-cli    │  docker buildx         │
│  │ (CI only)     │  --platform amd64      │
│  │               │  └── uses ROOT         │
│  │               │      Dockerfile ✅     │
│  │               │  docker push           │
│  └───────────────┘                        │
│          │                                │
│          ▼                                │
│  ┌───────────────┐                        │
│  │ Stage 3       │                        │
│  │ Deploy ECS    │                        │
│  │               │                        │
│  │ my-aws-cli    │  aws ecs               │
│  │ (CI only)     │  register-task-def     │
│  │               │  update-service        │
│  └───────────────┘                        │
└───────────────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────┐
│           Amazon ECR (Registry)            │
│                                           │
│  learnjenkinsapp:1.0.BUILD_ID              │
│  (linux/amd64 image) ✅                    │
└───────────────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────┐
│            Amazon ECS (Fargate)            │
│                                           │
│  Cluster: Learning-jenkins-app             │
│  Service: LearnJenkinsApp-Service-Prod    │
│                                           │
│  Task Definition (revision N)              │
│   ├── image from ECR ✅                    │
│   ├── port 80                              │
│   ├── awslogs enabled                     │
│   └── essential container                 │
│                                           │
│  Runtime container starts                 │
│  (THIS runs in production) ✅              │
└───────────────────────────────────────────┘

```