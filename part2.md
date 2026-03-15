# 🚀 TaskFlow — Part 2: Frontend

> You have a live backend behind an ALB. Now you will build the frontend in Next.js, containerize it, and deploy it to its own EC2 instance using the same CI/CD pattern you used in Part 1.

---

## Architecture Overview

```
GitHub (push to main)
        │
        ▼
GitHub Actions
  ├── Build Docker image
  ├── Push image to Amazon ECR
  └── SSH into EC2 (frontend)
        └── Pull image from ECR
            └── Run container on port 3000
                        │
                        ▼
            Application Load Balancer (port 80)
                        │
                        ▼
                  Public Traffic
                        │
                        ▼
        API calls → Backend ALB (port 80)
                        │
                        ▼
              EC2 (backend) port 4000
```

---

## What You Are Building

| Screen | Description |
|---|---|
| `/projects` | Lists all projects. User can create and delete projects. |
| `/projects/[id]` | Kanban board for a project. User can create, edit, and delete tasks across 4 columns. |

---

## Prerequisites

- Part 1 fully complete and verified
- Your backend ALB DNS name saved (e.g. `taskflow-api-alb-xxxx.us-east-1.elb.amazonaws.com`)
- Node.js v20+ installed locally
- pnpm → `npm install -g pnpm`

---

---

# PHASE 1 — Build the Frontend
Base on the last course you did on Udemy on learning Nextjs, you will build out a frontend application that will use the backend you deployed in the part 1.

## Step 1 — Initialize the Project

Scaffold a new Next.js project using the App Router with TypeScript and Tailwind CSS enabled. Create a new GitHub repository called `taskflow-frontend` and push the initial commit.

---

## Step 2 — Environment Variables

Create a `.env.local` file at the project root and add the `NEXT_PUBLIC_API_URL` variable pointing to your backend ALB DNS name.

Also create a `.env.example` file with the same variable but an empty value, and commit that file. The `.env.local` file must be added to `.gitignore` — never commit it.

> 💡 **Tip:** In Next.js, variables prefixed with `NEXT_PUBLIC_` are the only ones exposed to the browser. They are baked into the JavaScript bundle at **build time**, not at runtime. This means the API URL gets embedded into the app when Docker builds the image — not when the container starts. Keep this in mind when you get to the Dockerization phase.

---

## Step 3 — TypeScript Types

Before writing any components or API calls, define your TypeScript interfaces. Create types for `Project` and `Task` that match the shapes returned by the backend API.

Refer to the **API Reference** section at the bottom of this document to understand the exact shape of each response.

> 💡 **Tip:** Defining types first is not just good practice — it makes everything that comes after faster. Your editor will autocomplete fields, catch typos, and warn you when you use a response incorrectly before you even run the app.

---

## Step 4 — API Client

Install `axios` and create a centralized API client module. This module should:

- Create an axios instance with `baseURL` set to `process.env.NEXT_PUBLIC_API_URL`
- Export separate objects for `projectsApi` and `tasksApi`, each with functions for every CRUD operation (`getAll`, `getById`, `create`, `update`, `delete`)

Centralizing all API calls in one place means if the base URL or headers ever need to change, you only change them in one file.

---

## Step 5 — React Query Setup

Install `@tanstack/react-query` and `@tanstack/react-query-devtools`.

Create a `Providers` component that wraps the app in a `QueryClientProvider`. Update your root `layout.tsx` to use this provider. Also add `ReactQueryDevtools` — it gives you a visual inspector in development to see what data is cached, when it's fetching, and what queries are stale.

> 💡 **Tip:** React Query is not just a data fetching library — it is a server state manager. It handles caching, background refetching, loading states, and error states for you. Without it, you would have to manage all of that manually with `useEffect` and `useState`.

---

## Step 6 — Data Hooks

Create custom hooks that wrap your API calls with React Query. You should have two hook files — one for projects and one for tasks.

Each hook file should export individual hooks for every operation. For example, your projects hooks file should export `useProjects`, `useProject(id)`, `useCreateProject`, `useDeleteProject`. Your tasks hooks file should export `useTasks(projectId)`, `useCreateTask`, `useUpdateTask`, `useDeleteTask`.

Every mutation hook (`useCreateX`, `useUpdateX`, `useDeleteX`) must call `queryClient.invalidateQueries` in its `onSuccess` callback to keep the UI in sync after a change.

> 💡 **Tip:** `invalidateQueries` marks cached data as stale so React Query refetches it automatically. Without it, after creating or deleting an item, the UI would still show the old data until the user refreshed the page.

---

## Step 7 — Pages

### `/projects` — Projects List Page

This page should:
- Fetch and display all projects using your `useProjects` hook
- Show a loading spinner while data is being fetched
- Show an empty state message when there are no projects
- Have a button that opens a modal to create a new project
- Show each project as a card that links to `/projects/[id]`
- Have a delete button on each project card

### `/projects/[id]` — Project Detail Page

This page should:
- Read the `id` from the URL using `useParams`
- Fetch the project details with `useProject(id)`
- Fetch the project's tasks with `useTasks(id)`
- Display the project name and description at the top
- Render a `KanbanBoard` component passing the tasks and project ID
- Have a back link to `/projects`

---

## Step 8 — UI Components

Build the following reusable components. Keep them small and focused on a single responsibility.

**Spinner** — A centered loading indicator. Used on any page while data is being fetched.

**Badge** — A small colored label used to display a task's status and priority. Each status and priority value should have its own distinct color so they are easy to distinguish at a glance.

**Modal** — A generic overlay dialog. It should accept a `title`, an `open` boolean, an `onClose` callback, and `children`. Clicking the backdrop should close it. This component will be reused for both the project form and the task form.

---

## Step 9 — Project Components

**TaskCard** — Displays a single task inside a Kanban column. Should show the task title, description (truncated if long), priority badge, and due date. Edit and delete buttons should appear on hover.

**KanbanBoard** — Renders 4 columns: `To Do`, `In Progress`, `In Review`, `Done`. Filters the tasks array into the correct column based on each task's `status` field. Has an `+ Add Task` button that opens the task creation modal.

**TaskModal** — A form inside the `Modal` component for creating and editing tasks. Should handle both cases — when a `task` prop is passed it is in edit mode, when no task is passed it is in create mode. Fields: title (required), description (optional), status, priority, and due date.

**ProjectModal** — A form inside the `Modal` component for creating a new project. Fields: name (required) and description (optional).

---

## Step 10 — Forms and Validation

Install `react-hook-form`, `@hookform/resolvers`, and `zod`.

Define a Zod schema for the task form and another for the project form. Use `zodResolver` to connect each schema to its respective form. Validation error messages should appear inline below the relevant input field.

> 💡 **Tip:** Zod schemas double as TypeScript types using `z.infer<typeof schema>`. This means you define your validation rules once and get both runtime validation and compile-time type safety from the same object — no duplication.

---

## Step 11 — Test Locally Against the Live Backend

Run the development server and open `http://localhost:3000`. Test the complete flow:

- Projects list loads from the live backend
- Creating a project appears in the list immediately
- Opening a project shows its tasks in the Kanban board
- Creating, editing, and deleting tasks all work and update the UI instantly

Only move to Phase 2 once everything works correctly in development.

> 💡 **Tip:** Always verify the app works locally before containerizing it. Debugging a broken app inside a Docker container is significantly harder than debugging it with hot reload.

---

---

# PHASE 2 — AWS Infrastructure

You already did this for the backend in Part 1. You will now repeat the same pattern for the frontend — a new EC2 instance and a new ECR repository.

## Step 1 — Create an ECR Repository for the Frontend

1. Go to **Amazon ECR** in the AWS Console
2. Click **Repositories** → **Create repository**
3. Set visibility to **Private**
4. Repository name: `taskflow-frontend`
5. Leave all other settings as default and click **Create repository**
6. Click into the newly created repository and copy the **Repository URI** — it looks like:
   ```
   123456789012.dkr.ecr.us-east-1.amazonaws.com/taskflow-frontend
   ```
   Save this. You will need it in the GitHub Actions workflow.

## Step 2 — Launch a Frontend EC2 Instance

1. Go to **EC2** → **Launch instance**
2. Configure:

| Setting | Value |
|---|---|
| Name | `taskflow-frontend-server` |
| AMI | Amazon Linux 2023 |
| Instance type | `t2.micro` |
| Key pair | Use the same `taskflow-key.pem` from Part 1 |
| Auto-assign public IP | Enable |

3. Create a new security group named `taskflow-frontend-sg` with these inbound rules:

| Type | Port | Source | Purpose |
|---|---|---|---|
| SSH | 22 | Anywhere | So you can SSH in |
| Custom TCP | 3000 | Anywhere | App port (will be locked down after ALB is set up) |
| HTTP | 80 | Anywhere | ALB health checks |

## Step 3 — Attach the IAM Role to the Frontend EC2

Attach the same `taskflow-ec2-ecr-role` from Part 1:

1. Select your `taskflow-frontend-server` instance
2. **Actions** → **Security** → **Modify IAM role**
3. Select `taskflow-ec2-ecr-role` → **Update IAM role**

## Step 4 — Install Docker on the Frontend EC2

SSH in and install Docker — the exact same steps as Part 1:

```bash
ssh -i ~/Downloads/taskflow-key.pem ec2-user@YOUR_FRONTEND_EC2_IP

sudo apt update -y
sudo apt install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user

exit
# Log back in for the group change to take effect
ssh -i ~/Downloads/taskflow-key.pem ec2-user@YOUR_FRONTEND_EC2_IP
docker --version
```

---

---

# PHASE 3 — Dockerize the Frontend

## Step 1 — next.config.js

Add `output: "standalone"` to your Next.js config. This tells Next.js to produce a minimal self-contained build that includes only what is needed to run the app — no dev dependencies, no source files. It is specifically designed for Docker deployments.

## Step 2 — Dockerfile

Write a **multi-stage Dockerfile** with three stages:

**Stage 1 — deps:** Installs only production dependencies using a locked lockfile.

**Stage 2 — builder:** Copies node_modules from the deps stage, copies the rest of the source code, then runs `pnpm build`. This is also where `NEXT_PUBLIC_API_URL` must be declared as a build argument (`ARG`) and set as an environment variable (`ENV`) so Next.js can embed it into the bundle during the build.

**Stage 3 — runner:** Starts from a fresh `node:20-alpine` image and copies only the compiled output from the builder stage. Creates a non-root user to run the app. Exposes port `3000` and starts the app with `node server.js`.

> 💡 **Tip:** Multi-stage builds keep your final image small and clean. The production image contains no TypeScript compiler, no ESLint, no source files — just the compiled output. This makes it faster to pull, faster to start, and reduces the attack surface if the container is ever compromised.

## Step 3 — .dockerignore

Create a `.dockerignore` file to prevent unnecessary files from being sent to the Docker build context. At minimum, exclude `node_modules`, `.next`, `.git`, and `.env*.local`.

## Step 4 — Build and Test Locally

Build the image locally passing your backend ALB URL as a build argument. Run the container on port 3000 and open `http://localhost:3000` to confirm the containerized app connects to the live backend correctly before pushing anything to AWS.

> 💡 **Tip:** `NEXT_PUBLIC_*` variables must be passed as `--build-arg` during `docker build` — not as `-e` flags during `docker run`. If you pass them at runtime, the app will behave as if they are undefined because Next.js has already finished building.

---

---

# PHASE 4 — CI/CD Pipeline

You know this pattern from Part 1. The pipeline is nearly identical — the only differences are the ECR repository name, the container port, and the `--build-arg` for the API URL.

## Step 1 — Add GitHub Secrets

Go to your `taskflow-frontend` repo → **Settings** → **Secrets and variables** → **Actions**:

| Secret | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Same IAM user from Part 1 |
| `AWS_SECRET_ACCESS_KEY` | Same IAM user from Part 1 |
| `AWS_REGION` | `us-east-1` |
| `ECR_REPOSITORY` | `taskflow-frontend` |
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID |
| `EC2_HOST` | Public IP of `taskflow-frontend-server` |
| `EC2_USERNAME` | `ec2-user` |
| `EC2_SSH_KEY` | Full contents of `taskflow-key.pem` |
| `NEXT_PUBLIC_API_URL` | `http://YOUR_BACKEND_ALB_DNS` |

## Step 2 — CI Workflow (Pull Requests)

Create `.github/workflows/ci.yml`. This workflow should trigger on every pull request to `main` and run two jobs:

**Job 1 — Lint & Type Check:** Install dependencies, run `pnpm lint`, and run `pnpm tsc --noEmit`. This ensures no broken or poorly typed code can be merged.

**Job 2 — Docker Build Check:** Build the Docker image without pushing it, passing a placeholder value for `NEXT_PUBLIC_API_URL`. This confirms the Dockerfile itself is valid on every PR.

## Step 3 — Deploy Workflow (Merge to Main)

Create `.github/workflows/deploy.yml`. This workflow should trigger on every push to `main` and run two jobs in sequence:

**Job 1 — Build & Push:** Authenticate with AWS, log in to ECR, then build and push the Docker image tagged with both `latest` and the Git commit SHA (`github.sha`). The `NEXT_PUBLIC_API_URL` must be passed as a `--build-arg` here.

**Job 2 — Deploy (runs after Job 1):** SSH into the EC2 instance, authenticate Docker with ECR using the instance's IAM role, pull the latest image, stop and remove the old container, and start a new one on port 3000 with `--restart unless-stopped`. Clean up old images at the end.

> 💡 **Tip:** Tagging images with `github.sha` in addition to `latest` gives you a full history of every image that was ever deployed. If a deployment breaks production, you can immediately redeploy the previous `github.sha` tag without having to rebuild anything.

## Step 4 — Trigger the First Deployment

Commit and push your workflow files to `main`. Watch the **Actions** tab — both jobs should go green. Check the EC2 instance with `docker ps` to confirm the container is running.

---

---

# PHASE 5 — Application Load Balancer

Same pattern as Part 1 — Target Group → Load Balancer → lock the security group.

## Step 1 — Create a Target Group

1. Go to **EC2** → **Target Groups** → **Create target group**

| Setting | Value |
|---|---|
| Target type | Instances |
| Target group name | `taskflow-frontend-tg` |
| Protocol | HTTP |
| Port | `3000` |
| VPC | Default VPC |
| Health check protocol | HTTP |
| Health check path | `/` |

2. Click **Next** → select `taskflow-frontend-server` → **Include as pending below** → **Create target group**

## Step 2 — Create the Load Balancer

1. Go to **EC2** → **Load Balancers** → **Create load balancer** → **Application Load Balancer**

| Setting | Value |
|---|---|
| Name | `taskflow-frontend-alb` |
| Scheme | Internet-facing |
| VPC | Default VPC |
| Availability Zones | Select at least 2 subnets |

2. Create a new security group `taskflow-frontend-alb-sg` with one inbound rule: **HTTP, port 80, from Anywhere (0.0.0.0/0)**

3. Listener: **HTTP port 80** → forward to `taskflow-frontend-tg`

4. Click **Create load balancer**

## Step 3 — Lock Down the EC2 Security Group

Now that the ALB is in front, the EC2 should no longer accept traffic on port 3000 from the public internet:

1. Go to **Security Groups** → select `taskflow-frontend-sg`
2. Edit the inbound rule for port 3000
3. Change the source from `0.0.0.0/0` to the **security group ID** of `taskflow-frontend-alb-sg`
4. Save

> 💡 **Tip:** This is the same security pattern you applied to the backend in Part 1. The EC2 instance should never be directly reachable from the public internet on the application port. All traffic must flow through the ALB, which gives you a single controlled entry point.

## Step 4 — Verify the Full Stack

Wait 2-3 minutes for the ALB health checks to pass, then open the frontend ALB URL in your browser and confirm the complete end-to-end flow works — the frontend loads, the projects list fetches from the backend, and all CRUD operations work on the live deployment.

---

---

# Final Checklist

## Frontend Code
- [ ] App initializes and runs with `pnpm dev`
- [ ] `useTasks` hook built with `useTasks`, `useCreateTask`, `useUpdateTask`, `useDeleteTask`
- [ ] `ProjectModal` component built and working
- [ ] App fully tested locally against the live backend

## Dockerization
- [ ] `output: "standalone"` added to `next.config.js`
- [ ] Multi-stage Dockerfile written with `NEXT_PUBLIC_API_URL` as a build arg
- [ ] `.dockerignore` created
- [ ] `docker build` succeeds locally
- [ ] Container runs and connects to the live backend

## AWS Infrastructure
- [ ] ECR repository `taskflow-frontend` created
- [ ] `taskflow-frontend-server` EC2 instance running
- [ ] IAM role `taskflow-ec2-ecr-role` attached to the frontend EC2
- [ ] Docker installed on the frontend EC2

## CI/CD
- [ ] All 9 GitHub secrets added to `taskflow-frontend`
- [ ] `ci.yml` runs lint, type check, and Docker build on pull requests
- [ ] `deploy.yml` builds, pushes to ECR, and deploys to EC2 on merge to `main`
- [ ] Both pipelines pass green
- [ ] `docker ps` on the EC2 shows `taskflow-frontend` container running

## Load Balancer
- [ ] Target group `taskflow-frontend-tg` created with health check on `/`
- [ ] ALB `taskflow-frontend-alb` created and active
- [ ] EC2 security group updated — port 3000 only accepts traffic from the ALB security group

## End-to-End
- [ ] Frontend ALB URL loads the app in the browser
- [ ] Projects list shows data fetched from the backend
- [ ] All CRUD operations work on the live deployment
- [ ] `.env.local` is NOT committed to GitHub

---

## API Reference

Use these endpoints while building the frontend.

### Projects

| Method | Endpoint | Description |
|---|---|---|
| GET | `/projects` | List all projects |
| GET | `/projects/:id` | Get one project |
| POST | `/projects` | Create a project |
| PATCH | `/projects/:id` | Update a project |
| DELETE | `/projects/:id` | Delete project and its tasks |

**POST /projects body:** `{ "name": "string (required)", "description": "string (optional)" }`

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
  "title": "string (required)",
  "description": "string (optional)",
  "status": "TODO | IN_PROGRESS | IN_REVIEW | DONE",
  "priority": "LOW | MEDIUM | HIGH | URGENT",
  "projectId": "string (required)",
  "dueDate": "YYYY-MM-DD (optional)"
}
```

### Health Check

```
GET /health  →  { "status": "ok", "timestamp": "..." }
```

---

*Congratulations — you have a fully deployed, containerized, CI/CD-automated full-stack application on AWS.* 🚀