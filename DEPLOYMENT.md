# Docker & AWS Deployment Guide

Complete walkthrough of how this collaborative code editor was containerized with Docker and deployed to AWS using ECR, ECS Fargate, VPC, and an Application Load Balancer.

**Live URL:** http://docker-aws-yt-alb-790815787.ap-northeast-1.elb.amazonaws.com/

---

## Deployment Pipeline Overview

```
  Source Code (React + Node.js)
         │
         ▼
  ┌──────────────┐
  │  Docker Build │  ← Multi-stage build (frontend + backend in one image)
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │   AWS ECR     │  ← Private container registry (stores Docker images)
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │  ECS Fargate  │  ← Serverless container runtime (runs the image)
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │     ALB       │  ← Application Load Balancer (public URL + routing)
  └──────┬───────┘
         ▼
    🌍 Public URL
```

---

## Part 1: Docker — Containerizing the App

### What is Docker?

Docker packages your application + all its dependencies into a **container** — a lightweight, portable, isolated environment that runs the same everywhere (your laptop, AWS, any server).

### Why Docker?

- **"Works on my machine"** problem is solved — the container has everything it needs
- **Consistent environments** — dev, staging, production all run the same image
- **Easy deployment** — just push the image, pull and run anywhere
- **Isolation** — app doesn't conflict with host system packages

### The Dockerfile — Multi-Stage Build

```dockerfile
# ============ Stage 1: Build Frontend ============
FROM node:20-alpine as frontend-builder

COPY ./Frontend /app
WORKDIR /app
RUN npm install
RUN npm run build

# ============ Stage 2: Build Backend ============
FROM node:20-alpine

COPY ./Backend /app
WORKDIR /app
RUN npm install

# Copy compiled frontend into backend's public folder
COPY --from=frontend-builder /app/dist /app/public

CMD ["node", "server.js"]
```

### How the Multi-Stage Build Works

```
Stage 1: frontend-builder               Stage 2: Final Image
┌────────────────────────┐              ┌────────────────────────┐
│ node:20-alpine         │              │ node:20-alpine         │
│                        │              │                        │
│ COPY Frontend → /app   │              │ COPY Backend → /app    │
│ npm install            │              │ npm install            │
│ npm run build          │              │                        │
│         │              │              │ COPY dist → /app/public│
│         ▼              │───dist/──────▶│                        │
│  /app/dist/ (built JS) │              │ CMD node server.js     │
│                        │              │                        │
│ ❌ DISCARDED after     │              │ ✅ FINAL IMAGE         │
│    build completes     │              │    (~200MB vs ~800MB)  │
└────────────────────────┘              └────────────────────────┘
```

**Why multi-stage?**

- Stage 1 builds the React app → produces static files in `dist/`
- Stage 2 takes only the `dist/` output and the backend code
- All the frontend `node_modules`, source files, build tools are **discarded**
- Final image is **much smaller** — only contains what's needed to run

### Key Details

| Aspect           | Value                                                     |
| ---------------- | --------------------------------------------------------- |
| Base image       | `node:20-alpine` (lightweight ~180MB vs ~900MB full node) |
| Frontend output  | `dist/` folder → static HTML, JS, CSS                     |
| Backend serves   | `express.static("public")` → the frontend build           |
| Exposed port     | `3000` (inside the container)                             |
| Single container | Both frontend + backend in one image                      |

### Docker Commands Used

```bash
# Build for ARM Mac → AMD64 Linux (required for AWS Fargate)
docker buildx build --platform linux/amd64 -t docker_aws/server --load .

# Regular build (if on x86/AMD64 machine)
docker build -t docker_aws/server .

# Test locally
docker run -p 3000:3000 docker_aws/server
```

> **Important:** AWS Fargate runs on `linux/amd64`. If you're building on an ARM Mac (M1/M2/M3), you MUST use `--platform linux/amd64` or the container will crash on AWS.

---

## Part 2: AWS ECR — Container Registry

### What is ECR?

**Elastic Container Registry (ECR)** is AWS's private Docker image registry — like Docker Hub, but private and integrated with AWS services.

### Why ECR?

- **Private** — your images aren't public
- **Integrated with ECS** — ECS can pull images directly from ECR without extra config
- **IAM-based access** — secure, no separate credentials needed
- **Region-local** — fast pulls since ECR and ECS are in the same AWS region

### ECR Commands Used

```bash
# Step 1: Authenticate Docker to your ECR registry
aws ecr get-login-password --region ap-northeast-1 | \
  docker login --username AWS --password-stdin \
  102783063058.dkr.ecr.ap-northeast-1.amazonaws.com
```

**What this does:**

- `aws ecr get-login-password` → gets a temporary auth token from AWS
- Pipes it to `docker login` → authenticates your local Docker client with ECR
- Token expires in 12 hours — you'll need to re-run this for future pushes

```bash
# Step 2: Tag the image with the ECR repository URI
docker tag docker_aws/server:latest \
  102783063058.dkr.ecr.ap-northeast-1.amazonaws.com/docker_aws/server:latest
```

**What this does:**

- Docker images need to be named with the full ECR URI for push to work
- Format: `<account-id>.dkr.ecr.<region>.amazonaws.com/<repo-name>:<tag>`

```bash
# Step 3: Push image to ECR
docker push 102783063058.dkr.ecr.ap-northeast-1.amazonaws.com/docker_aws/server:latest
```

**What this does:**

- Uploads the Docker image layers to your ECR repository
- Only pushes layers that have changed (incremental)

### ECR Flow

```
Your Machine                          AWS ECR
┌──────────────┐    docker push     ┌──────────────────────┐
│ docker_aws/  │ ────────────────▶  │ 102783063058.dkr.ecr │
│ server:latest│                    │ .ap-northeast-1      │
│              │                    │ .amazonaws.com/      │
│ (local image)│                    │ docker_aws/server    │
└──────────────┘                    │ :latest              │
                                    └──────────┬───────────┘
                                               │
                                    ECS pulls from here
```

---

## Part 3: AWS ECS — Running the Container

### What is ECS?

**Elastic Container Service (ECS)** is AWS's container orchestration service. It manages running, scaling, and monitoring your Docker containers.

### What is Fargate?

**Fargate** is ECS's **serverless compute engine** — you don't manage any EC2 instances. AWS handles the servers entirely. You just say "run this container" and it runs.

### Why ECS + Fargate?

| Feature      | Benefit                                             |
| ------------ | --------------------------------------------------- |
| No servers   | No EC2 instances to manage, patch, or scale         |
| Pay per use  | Billed only for vCPU and memory your container uses |
| Auto-scaling | Can scale containers based on CPU/memory/requests   |
| Integrated   | Works natively with ECR, ALB, CloudWatch, IAM       |

### ECS Concepts — How They Fit Together

```
┌─────────────────────────────────────────────────────┐
│                   ECS Cluster                        │
│         (logical grouping of services)               │
│                                                      │
│  ┌───────────────────────────────────────────────┐  │
│  │                 Service                         │  │
│  │    (keeps desired number of tasks running)      │  │
│  │                                                 │  │
│  │  ┌─────────────────────────────────────────┐   │  │
│  │  │          Task (Fargate)                  │   │  │
│  │  │                                          │   │  │
│  │  │  ┌────────────────────────────────────┐  │   │  │
│  │  │  │  Container: docker_aws/server      │  │   │  │
│  │  │  │  Port: 3000                        │  │   │  │
│  │  │  │  Image: from ECR                   │  │   │  │
│  │  │  └────────────────────────────────────┘  │   │  │
│  │  └─────────────────────────────────────────┘   │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### Step-by-Step: What Was Created

#### 1. Task Definition

The **task definition** is a blueprint — like a `docker-compose.yml` for AWS. It tells ECS:

| Setting         | Value                 | Why                                          |
| --------------- | --------------------- | -------------------------------------------- |
| OS/Architecture | Linux/X86_64          | Matches the `linux/amd64` Docker build       |
| Container name  | (your container name) | Identifier for the container in this task    |
| Image URI       | ECR image URI         | Where to pull the Docker image from          |
| Container port  | 3000                  | Port the app listens on inside the container |
| CPU/Memory      | (configured)          | Resource limits for the Fargate task         |

#### 2. IAM Roles (Permissions)

Two IAM roles were needed:

```
┌──────────────────────────────────────────────────────────┐
│              IAM Roles for ECS                            │
│                                                           │
│  Task Execution Role                Task Role             │
│  ─────────────────────              ──────────            │
│  Used BY ECS to:                    Used BY your app to:  │
│  • Pull image from ECR             • Call AWS services    │
│  • Send logs to CloudWatch         • Access S3, DynamoDB  │
│  • Retrieve secrets                  (if needed)          │
│                                                           │
│  Policy: AmazonECSTaskExecutionRolePolicy                │
│  Trust: ecs-tasks.amazonaws.com                           │
└──────────────────────────────────────────────────────────┘
```

**How the roles were set up:**

1. In the **main/root AWS account** → IAM → Create Role
2. AWS Service → Use case: **ECS** → **Elastic Container Service Task**
3. Attach policy: `AmazonECSTaskExecutionRolePolicy`
4. In the **Docker_AWS service account** → select these roles in the Task Definition

#### 3. ECS Cluster

A **cluster** is a logical grouping. Think of it as a namespace — it holds one or more services.

- Cluster created in the **Docker_AWS account**
- No servers to configure (Fargate handles compute)

#### 4. Service (Created from Task Definition)

The service:

- Runs the task definition
- Maintains **desired count** of running tasks (e.g., 1)
- Restarts tasks if they crash
- Connects to VPC, security group, and load balancer

---

## Part 4: VPC — Networking

### What is a VPC?

**Virtual Private Cloud (VPC)** is your own isolated network in AWS. Everything (ECS tasks, load balancers, databases) runs inside a VPC.

### What was configured?

```
┌─────────────────────────────────────────────────┐
│                     VPC                          │
│            (created in main root account)         │
│                                                   │
│  ┌────────────────┐    ┌────────────────┐        │
│  │  Public Subnet  │    │  Public Subnet  │        │
│  │  (AZ-1)         │    │  (AZ-2)         │        │
│  │                  │    │                  │        │
│  │  ECS Task runs  │    │  ALB routes      │        │
│  │  here            │    │  here too        │        │
│  └────────────────┘    └────────────────┘        │
│                                                   │
│  ❌ Private subnets removed (not needed for this) │
└─────────────────────────────────────────────────┘
```

### Why 2 Public Subnets?

- ALB **requires at least 2 subnets** in different Availability Zones
- Both are **public** (have internet access via Internet Gateway)
- Fargate tasks run in public subnets with **public IP** enabled
- Private subnets were removed — not needed for this simple setup

---

## Part 5: Security Group — Firewall

### What is a Security Group (SG)?

A **security group** is a virtual firewall that controls inbound/outbound traffic to your resources.

### What was configured?

| Rule     | Type   | Port | Source    | Why                                    |
| -------- | ------ | ---- | --------- | -------------------------------------- |
| Inbound  | HTTP   | 80   | 0.0.0.0/0 | Allow public web traffic to ALB        |
| Inbound  | Custom | 3000 | SG self   | Allow ALB to reach container port 3000 |
| Outbound | All    | All  | 0.0.0.0/0 | Allow container to pull images, etc.   |

**Created in:** Main root account → EC2 → Security Groups
**Then selected** in the Docker_AWS ECS Service configuration

---

## Part 6: ALB — Application Load Balancer

### What is an ALB?

**Application Load Balancer (ALB)** distributes incoming traffic to your containers. It gives you a **public DNS URL** to access your app.

### Why ALB?

- Fargate tasks don't have a fixed public IP — ALB provides a **stable endpoint**
- **Health checks** — ALB pings `/health` endpoint to ensure container is alive
- **Distributes traffic** — if you scale to multiple tasks, ALB balances between them
- **WebSocket support** — ALB supports the WebSocket connections that Socket.IO needs

### How ALB Connects to ECS

```
Internet
   │
   ▼
┌──────────────────────────────────────────┐
│  ALB (Application Load Balancer)          │
│  DNS: docker-aws-yt-alb-790815787        │
│       .ap-northeast-1.elb.amazonaws.com   │
│                                           │
│  Listener: Port 80 (HTTP)                 │
│       │                                   │
│       ▼                                   │
│  Target Group                             │
│  ┌─────────────────────────────────────┐ │
│  │ Health Check: GET /health → 200 OK  │ │
│  │ Target: ECS Fargate tasks           │ │
│  │ Port: 3000                          │ │
│  └──────────┬──────────────────────────┘ │
└─────────────┼────────────────────────────┘
              │
              ▼
┌─────────────────────────┐
│  ECS Fargate Task        │
│  Container: port 3000    │
│                          │
│  Express → static files  │
│  Socket.IO → WebSocket   │
│  /health → 200 OK        │
└─────────────────────────┘
```

### ALB Setup Details

| Setting           | Value                             | Why                                       |
| ----------------- | --------------------------------- | ----------------------------------------- |
| Type              | Application Load Balancer         | Layer 7 — understands HTTP, WebSocket     |
| Scheme            | Internet-facing                   | Needs to be publicly accessible           |
| Listener port     | 80 (HTTP)                         | Standard web traffic port                 |
| Target type       | IP                                | Fargate tasks register by IP (not EC2 ID) |
| Target port       | 3000                              | The port your container listens on        |
| Health check path | `/health`                         | Express endpoint returns `200 OK`         |
| Subnets           | 2 public subnets in different AZs | ALB needs multi-AZ for high availability  |

### Where ALB Was Created

- Can be created either:
  - **In the main root account** → EC2 → Load Balancers → Create ALB
  - **Or directly in the ECS Service creation** wizard (it offers to create one)
- Both approaches work — the ECS service then registers its tasks with the ALB's target group

---

## Full Architecture Diagram

```
                         ┌──────────────────┐
                         │    Internet       │
                         │    (Users)        │
                         └────────┬─────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────┐
│                          VPC                                 │
│                                                              │
│          ┌──────────────────────────────────┐                │
│          │    ALB (Internet-facing)          │                │
│          │    Port 80 → Target Group        │                │
│          │    docker-aws-yt-alb-...         │                │
│          │    .ap-northeast-1.elb           │                │
│          │    .amazonaws.com                │                │
│          └──────────┬───────────────────────┘                │
│                     │                                        │
│     ┌───────────────┼────────────────┐                       │
│     │               │                │                       │
│     ▼               ▼                                        │
│  ┌──────────┐  ┌──────────┐                                  │
│  │ Public   │  │ Public   │     ◀── 2 AZs for availability  │
│  │ Subnet 1 │  │ Subnet 2 │                                  │
│  │ (AZ-a)   │  │ (AZ-c)   │                                  │
│  └────┬─────┘  └──────────┘                                  │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────────────────────────────────┐                     │
│  │        ECS Cluster                   │                    │
│  │                                      │                    │
│  │  ┌──────────────────────────────┐   │                    │
│  │  │  Service (Fargate)            │   │                    │
│  │  │                               │   │                    │
│  │  │  ┌────────────────────────┐  │   │                    │
│  │  │  │ Task                    │  │   │                    │
│  │  │  │ ┌────────────────────┐ │  │   │                    │
│  │  │  │ │ Container          │ │  │   │                    │
│  │  │  │ │ Image: from ECR   │ │  │   │                    │
│  │  │  │ │ Port: 3000        │ │  │   │                    │
│  │  │  │ └────────────────────┘ │  │   │                    │
│  │  │  └────────────────────────┘  │   │                    │
│  │  └──────────────────────────────┘   │                    │
│  └─────────────────────────────────────┘                     │
│                                                              │
│  Security Group: Allow 80 (public) + 3000 (ALB→task)        │
│                                                              │
└──────────────────────────────────────────────────────────────┘

        ┌──────────────────────────────┐
        │        AWS ECR               │
        │  102783063058.dkr.ecr        │
        │  .ap-northeast-1             │
        │  .amazonaws.com/             │
        │  docker_aws/server:latest    │
        │                              │
        │  (ECS pulls image from here) │
        └──────────────────────────────┘
```

---

## Redeployment — How to Update

When you make code changes:

```bash
# 1. Rebuild the Docker image (from project root)
docker buildx build --platform linux/amd64 -t docker_aws/server --load .

# 2. Re-authenticate with ECR (token expires every 12 hours)
aws ecr get-login-password --region ap-northeast-1 | \
  docker login --username AWS --password-stdin \
  102783063058.dkr.ecr.ap-northeast-1.amazonaws.com

# 3. Tag with ECR URI
docker tag docker_aws/server:latest \
  102783063058.dkr.ecr.ap-northeast-1.amazonaws.com/docker_aws/server:latest

# 4. Push new image to ECR
docker push \
  102783063058.dkr.ecr.ap-northeast-1.amazonaws.com/docker_aws/server:latest

# 5. Force ECS to pull new image and redeploy
aws ecs update-service \
  --cluster <your-cluster-name> \
  --service <your-service-name> \
  --force-new-deployment \
  --region ap-northeast-1
```

> ECS will perform a **rolling deployment** — starts new task with new image, routes traffic to it, then stops old task. Zero downtime.

---

## Troubleshooting Tips

| Problem                     | What to Check                                          |
| --------------------------- | ------------------------------------------------------ |
| Container keeps restarting  | Check ECS → Task → Logs (CloudWatch)                   |
| ALB returns 502/503         | Health check failing — verify `/health` endpoint works |
| Users can't see each other  | Frontend connecting to `localhost` instead of ALB URL  |
| Image push fails            | Re-run `ecr get-login-password` (token expired)        |
| Task won't start            | Check Task Execution Role has ECR pull permissions     |
| Cross-platform build issues | Use `--platform linux/amd64` on ARM Macs               |

---

## Key Learnings

What this deployment covers — real DevOps/fullstack skills:

| Skill               | What You Learned                                         |
| ------------------- | -------------------------------------------------------- |
| **Docker**          | Multi-stage builds, cross-platform builds, image tagging |
| **ECR**             | Private registry, authentication, image versioning       |
| **ECS**             | Task definitions, services, clusters                     |
| **Fargate**         | Serverless containers — no EC2 management                |
| **IAM**             | Roles for task execution, cross-account patterns         |
| **VPC**             | Subnets, AZs, internet gateway                           |
| **Security Groups** | Inbound/outbound rules, port management                  |
| **ALB**             | Load balancing, health checks, target groups, WebSocket  |
| **Networking**      | How traffic flows: Internet → ALB → Container            |
| **CI/CD (manual)**  | Build → Push → Deploy pipeline                           |
