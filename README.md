# 🚀 TaskFlow — Project Guide

> A task management app. This guide is split into two parts. **Complete Part 1 fully before starting Part 2.**

---

## What You Are Building

**TaskFlow** is a task management app with two screens:

- A **Projects** page that lists all projects
- A **Project Detail** page that shows tasks in a Kanban board (To Do / In Progress / In Review / Done)

---

## Your Responsibilities

| Task | Who |
|---|---|
| Backend API (Express.js) | ✅ Already built |
| Deploy the backend to AWS EC2 | 👉 You |
| Build the Next.js frontend | 👉 You |
| Dockerize the frontend | 👉 You |
| CI/CD pipeline with GitHub Actions | 👉 You |
| Deploy the frontend to AWS EC2 | 👉 You |

---

## Tech Stack

| | Technology |
|---|---|
| Frontend | Next.js 14+, TypeScript, Tailwind CSS |
| Data Fetching | TanStack Query |
| Forms | React Hook Form + Zod |
| Containerization | Docker |
| Container Registry | Amazon ECR |
| Hosting | AWS EC2 |
| Load Balancer | AWS Application Load Balancer |
| CI/CD | GitHub Actions |

---

# PART 1: Deploy the Backend to AWS

The backend is a pre-built Express.js REST API. Your job is to deploy it on AWS so it is publicly reachable via a Load Balancer URL.

**Backend repo:** `https://github.com/royalteegee/taskflow-api`

Fork this repo into your own GitHub account before starting. All CI/CD will run from your fork.

---

## Architecture Overview

Here is what you will build in Part 1:

```
GitHub (push to main)
        │
        ▼
GitHub Actions
  ├── Build Docker image
  ├── Push image to Amazon ECR
  └── SSH into EC2
        └── Pull image from ECR
            └── Run container on port 4000
                        │
                        ▼
            Application Load Balancer (port 80)
                        │
                        ▼
                  Public Traffic
```

---

## Step 1 — Create an ECR Repository

Amazon ECR is a private Docker registry hosted on AWS. This is where GitHub Actions will push your built images.

**In the AWS Console:**

1. Go to **Amazon ECR** → **Repositories** → **Create repository**
2. Set visibility to **Private**
3. Repository name: `taskflow-api`
4. Leave all other settings as default and click **Create repository**
5. Copy the **Repository URI** — it looks like:
   ```
   123456789012.dkr.ecr.us-east-1.amazonaws.com/taskflow-api
   ```
   Save this. You will need it in the GitHub Actions workflow.

---

## Step 2 — Launch an EC2 Instance

This is the server that will run your Docker container.

**In the AWS Console:**

1. Go to **EC2** → **Instances** → **Launch instance**
2. Configure it as follows:

| Setting | Value |
|---|---|
| Name | `taskflow-api-server` |
| AMI | ubuntu |
| Instance type | `t2.micro` |
| Key pair | Create a new key pair → name it `taskflow-key` → download the `.pem` file |
| Auto-assign public IP | Enable |

3. Under **Network settings** → **Create security group** — name it `taskflow-api-sg` and add these inbound rules:

| Type | Port | Source | Purpose |
|---|---|---|---|
| SSH | 22 | My IP | So you can SSH in |
| Custom TCP | 4000 | Anywhere (0.0.0.0/0) | API port (will be restricted later via ALB) |
| HTTP | 80 | Anywhere (0.0.0.0/0) | Load balancer health checks |

4. Click **Launch instance**

> 💡 **Tip:** The `.pem` key file is your only way to SSH into this server. If you lose it, you cannot recover it — you would have to create a new instance. Store it somewhere safe and never commit it to GitHub.

---

## Step 3 — Attach an IAM Role to the EC2 Instance

The EC2 instance needs permission to pull images from ECR. You do this by attaching an IAM Role — not by putting credentials on the server.

**Create the role:**

1. Go to **IAM** → **Roles** → **Create role**
2. Trusted entity: **AWS service** → **EC2**
3. Attach the policy: `AmazonEC2ContainerRegistryReadOnly`
4. Role name: `taskflow-ec2-ecr-role`
5. Click **Create role**

**Attach the role to your EC2 instance:**

1. Go to **EC2** → **Instances** → select `taskflow-api-server`
2. Click **Actions** → **Security** → **Modify IAM role**
3. Select `taskflow-ec2-ecr-role` and click **Update IAM role**

> 💡 **Tip:** IAM Roles are the correct way to give AWS services permission to talk to each other. Never paste your AWS Access Key and Secret onto an EC2 instance — if the server is compromised, those credentials leak. A role is temporary and scoped to only what the instance needs.

---

## Step 4 — Install Docker on the EC2 Instance

SSH into your new instance and install Docker.

```bash
# Replace with your actual public IP from the EC2 console
# Replace with the path to your downloaded .pem file
chmod 400 ~/Downloads/taskflow-key.pem

ssh -i ~/Downloads/taskflow-key.pem ec2-user@YOUR_EC2_PUBLIC_IP
```

Once inside the instance, run:

```bash
# Update packages
sudo apt update -y

# Install Docker
sudo apt install -y docker

# Start the Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Allow ec2-user to run docker without sudo
sudo usermod -aG docker ubuntu

# Verify Docker is running
docker --version
```

Log out and back in for the group change to take effect:

```bash
exit
ssh -i ~/Downloads/taskflow-key.pem ec2-user@YOUR_EC2_PUBLIC_IP
```

---

## Step 5 — Create a GitHub Actions IAM User

GitHub Actions needs AWS credentials to push images to ECR. Create a dedicated IAM user with the minimum permissions required.

1. Go to **IAM** → **Users** → **Create user**
2. Username: `github-actions-taskflow`
3. On the permissions page, attach these policies directly:
   - `AmazonEC2ContainerRegistryPowerUser` — allows building and pushing images to ECR
4. After creating the user, go to **Security credentials** → **Create access key**
5. Choose **Application running outside AWS**, click through, and **download the CSV**

> 💡 **Tip:** This IAM user should only have the permissions it actually needs — nothing more. This is called the Principle of Least Privilege. If this user's credentials are ever leaked, the blast radius is limited to ECR operations only.

---

## Step 6 — Add GitHub Secrets

Go to your forked `taskflow-api` repo on GitHub → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Add all of the following:

| Secret Name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | From the IAM user CSV you downloaded |
| `AWS_SECRET_ACCESS_KEY` | From the IAM user CSV you downloaded |
| `AWS_REGION` | `us-east-1` |
| `ECR_REPOSITORY` | `taskflow-api` |
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID (find it in the top-right of the AWS console) |
| `EC2_HOST` | The public IP address of your EC2 instance |
| `EC2_USERNAME` | `ec2-user` |
| `EC2_SSH_KEY` | The **full contents** of your `taskflow-key.pem` file |

To get the full contents of your `.pem` file for the `EC2_SSH_KEY` secret:

```bash
cat ~/Downloads/taskflow-key.pem
```

Copy everything including the `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----` lines.

---

## Step 7 — Create the CI/CD Pipeline

Create the file `.github/workflows/deploy.yml` inside the `taskflow-api` repo:

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

env:
  AWS_REGION: ${{ secrets.AWS_REGION }}
  ECR_REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com
  ECR_REPOSITORY: ${{ secrets.ECR_REPOSITORY }}
  IMAGE_TAG: ${{ github.sha }}

jobs:
  build-and-push:
    name: Build & Push to ECR
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push Docker image
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

  deploy:
    name: Deploy to EC2
    runs-on: ubuntu-latest
    needs: build-and-push

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          key: ${{ secrets.EC2_SSH_KEY }}
          envs: ECR_REGISTRY,ECR_REPOSITORY,IMAGE_TAG,AWS_REGION
          script: |
            # Authenticate EC2 with ECR (uses the IAM role attached to the instance)
            aws ecr get-login-password --region $AWS_REGION | \
              docker login --username AWS --password-stdin $ECR_REGISTRY

            # Pull the latest image
            docker pull $ECR_REGISTRY/$ECR_REPOSITORY:latest

            # Stop and remove the old container if it exists
            docker stop taskflow-api || true
            docker rm taskflow-api || true

            # Run the new container
            docker run -d \
              --name taskflow-api \
              --restart unless-stopped \
              -p 4000:4000 \
              $ECR_REGISTRY/$ECR_REPOSITORY:latest

            # Clean up old images to save disk space
            docker image prune -f
```

**How this pipeline works:**

1. **`build-and-push` job** — Every push to `main` triggers a Docker build. The image is tagged with the Git commit SHA (for traceability) and also as `latest`. Both tags are pushed to ECR.

2. **`deploy` job** — Runs after `build-and-push` succeeds (`needs: build-and-push`). It SSHes into the EC2 instance, pulls the new image from ECR, stops the old container, and starts a fresh one on port 4000. The `--restart unless-stopped` flag means the container will automatically restart if the EC2 instance reboots.

> 💡 **Tip:** Tagging images with the Git commit SHA (`github.sha`) is a best practice. It means every image in ECR maps exactly to a commit in GitHub. If something breaks in production, you can look at the image tag to know precisely what code is running — and roll back to a previous tag instantly.

---

## Step 8 — Trigger the First Deployment

Push any change to the `main` branch of your forked repo to trigger the pipeline:

```bash
# Inside your local clone of taskflow-api
git commit --allow-empty -m "ci: trigger first deployment"
git push origin main
```

Go to the **Actions** tab of your GitHub repo and watch the pipeline run. Both jobs should go green. If either fails, click into the job to read the error output.

---

## Step 9 — Verify the Backend is Live

Once the pipeline is green, test that the API is running on your EC2 instance:

```bash
# Health check
curl http://YOUR_EC2_PUBLIC_IP:4000/health
# Expected: { "status": "ok", "timestamp": "..." }

# List projects
curl http://YOUR_EC2_PUBLIC_IP:4000/projects
# Expected: [ { "id": "proj-001", "name": "Website Redesign", ... } ]
```

You can also SSH in and check the container status directly:

```bash
ssh -i ~/Downloads/taskflow-key.pem ec2-user@YOUR_EC2_PUBLIC_IP

# Check the container is running
docker ps

# View live logs
docker logs taskflow-api -f
```

> 💡 **Tip:** `docker ps` shows running containers. `docker logs <name>` shows the container's stdout — this is where your `console.log` and error messages appear. In a real production setup you would ship these logs to CloudWatch, but direct container logs are fine for this project.

---

## Step 10 — Create an Application Load Balancer

Right now the API is reachable directly on `http://EC2_IP:4000`. In production you never expose raw EC2 instances to the internet — you put a Load Balancer in front. The ALB will:

- Accept traffic on port **80** (standard HTTP)
- Forward it to the EC2 instance on port **4000**
- Perform health checks to confirm the container is running
- Give you a stable DNS name that doesn't change even if you replace the EC2 instance

### Create a Target Group

A Target Group tells the ALB which EC2 instances to send traffic to.

1. Go to **EC2** → **Target Groups** → **Create target group**
2. Configure as follows:

| Setting | Value |
|---|---|
| Target type | Instances |
| Target group name | `taskflow-api-tg` |
| Protocol | HTTP |
| Port | `4000` |
| VPC | Select your default VPC |
| Health check protocol | HTTP |
| Health check path | `/health` |

3. Click **Next** → select your `taskflow-api-server` instance → click **Include as pending below** → **Create target group**

### Create the Load Balancer

1. Go to **EC2** → **Load Balancers** → **Create load balancer**
2. Choose **Application Load Balancer**
3. Configure as follows:

| Setting | Value |
|---|---|
| Name | `taskflow-api-alb` |
| Scheme | Internet-facing |
| IP address type | IPv4 |
| VPC | Default VPC |
| Availability Zones | Select at least 2 subnets |

4. Under **Security groups** — create a new security group named `taskflow-alb-sg` with one inbound rule: **HTTP, port 80, from Anywhere (0.0.0.0/0)**

5. Under **Listeners and routing**:
   - Protocol: **HTTP**, Port: **80**
   - Default action: **Forward to** → select `taskflow-api-tg`

6. Click **Create load balancer**

### Update the EC2 Security Group

Now that the ALB is in front of the EC2 instance, tighten the EC2 security group so port 4000 is only reachable from the ALB — not the open internet.

1. Go to **EC2** → **Security Groups** → select `taskflow-api-sg`
2. Edit the inbound rule for port 4000:
   - Change **Source** from `0.0.0.0/0` to the **security group ID** of `taskflow-alb-sg`
3. Save the rule

> 💡 **Tip:** This is a critical security step. After adding the ALB, the EC2 instance should no longer accept traffic from the public internet on port 4000 — only from the ALB's security group. This way the ALB is the single entry point, and you can add rate limiting, WAF rules, and HTTPS certificates there without touching the application.

### Verify the Load Balancer

Wait 2-3 minutes for the ALB to provision and the health checks to pass. Find your ALB DNS name in the console — it looks like:

```
taskflow-api-alb-1234567890.us-east-1.elb.amazonaws.com
```

Test it:

```bash
curl http://taskflow-api-alb-1234567890.us-east-1.elb.amazonaws.com/health
# Expected: { "status": "ok", "timestamp": "..." }

curl http://taskflow-api-alb-1234567890.us-east-1.elb.amazonaws.com/projects
# Expected: [ { "id": "proj-001", "name": "Website Redesign", ... } ]
```

If both return data, the backend is fully deployed. Save this ALB DNS name — it is the `NEXT_PUBLIC_API_URL` you will use throughout Part 2.

---

## Part 1 Checklist

Before moving on to Part 2, confirm every item below:

- [ ] ECR repository `taskflow-api` created in AWS
- [ ] EC2 instance `taskflow-api-server` launched and running
- [ ] IAM role `taskflow-ec2-ecr-role` attached to the EC2 instance
- [ ] Docker installed on the EC2 instance
- [ ] IAM user `github-actions-taskflow` created with ECR permissions
- [ ] All 8 GitHub secrets added to the repo
- [ ] `.github/workflows/deploy.yml` created and committed
- [ ] GitHub Actions pipeline ran successfully (both jobs green)
- [ ] `docker ps` on EC2 shows `taskflow-api` container running
- [ ] `curl http://EC2_IP:4000/health` returns `{ "status": "ok" }`
- [ ] Target group `taskflow-api-tg` created with health check on `/health`
- [ ] Application Load Balancer `taskflow-api-alb` created and active
- [ ] EC2 security group updated — port 4000 only accepts traffic from the ALB security group
- [ ] `curl http://ALB_DNS/projects` returns project data
- [ ] ALB DNS name saved — this is your backend URL for Part 2

---

## API Reference

Use these endpoints while building the frontend in Part 2.

### Projects

| Method | Endpoint | Description |
|---|---|---|
| GET | `/projects` | List all projects |
| GET | `/projects/:id` | Get one project |
| POST | `/projects` | Create a project |
| PATCH | `/projects/:id` | Update a project |
| DELETE | `/projects/:id` | Delete project and its tasks |

**POST /projects body:**
```json
{ "name": "My Project", "description": "Optional" }
```

### Tasks

| Method | Endpoint | Description |
|---|---|---|
| GET | `/tasks?projectId=xxx` | List tasks for a project |
| GET | `/tasks/:id` | Get one task |
| POST | `/tasks` | Create a task |
| PATCH | `/tasks/:id` | Update a task |
| DELETE | `/tasks/:id` | Delete a task |

**POST /tasks body:**
```json
{
  "title": "My Task",
  "status": "TODO",
  "priority": "MEDIUM",
  "projectId": "proj-001",
  "description": "Optional",
  "dueDate": "2024-03-01"
}
```

**Valid status values:** `TODO` | `IN_PROGRESS` | `IN_REVIEW` | `DONE`

**Valid priority values:** `LOW` | `MEDIUM` | `HIGH` | `URGENT`

### Health Check

```
GET /health  →  { "status": "ok", "timestamp": "..." }
```

---

*Part 2 (Frontend) will be provided once Part 1 is complete and verified.* 🚀