# FlowBoard — Team Task Management

A full-stack task management web app (mini-Jira) built with **React**, **Node.js/Express**, and **AWS** (DynamoDB, S3, Cognito, SNS, SQS, Lambda, CloudWatch).
Test Credinntials
Video link: https://drive.google.com/file/d/1NYzDwbNlVBScTshrKkT8Xk-yFxeTCtHS/view?usp=sharing

Manager
email:kiromanager@gmail.com
pass:Password1!

Employee
email:kiroreda750@gmail.com
pass:1234!Pass

---

## Architecture Overview
<img width="4540" height="3916" alt="architecture" src="https://github.com/user-attachments/assets/14199fcf-a0dd-49a9-b79a-078a8b1ad057" />



# FlowBoard — Architecture Documentation

**FlowBoard (mini-Jira)** is a full-stack team task management application built on **React**, **Node.js/Express**, and a highly available **AWS** infrastructure spanning two Availability Zones.

---

## 1. System Overview

FlowBoard follows a classic three-tier, event-driven AWS architecture:

- **Edge/Delivery Layer** — CloudFront serves the React SPA and static assets to end users.
- **Application Layer** — A load-balanced, auto-scaled fleet of EC2 instances runs the Node.js/Express API across two Availability Zones for high availability.
- **Data & Integration Layer** — DynamoDB (data), S3 (files/images), Cognito (auth), and an SNS → SQS → Lambda pipeline (async notifications and image processing).

The design emphasizes **high availability** (multi-AZ, load-balanced, auto-scaled), **asynchronous decoupling** (SNS/SQS/Lambda for non-critical side effects), and **server-side authorization** (team isolation enforced in the API, not the client).

---

## 2. Architecture Diagram

The reference diagram ("Horizontal AWS HA Architecture") shows the following structure:

```
End User
   │
   ▼
Amazon CloudFront  ──────────────────────────────────────────────┐
   │                                                              │
   ▼                                                              │
Custom VPC                                                        │
├── Availability Zone 1                                           │
│    ├── Public Subnet 1                                          │
│    │     ├── Application Load Balancer                          │
│    │     └── NAT Gateway                                        │
│    └── Private Subnet 1 (Auto Scaling Group)                    │
│          └── Amazon EC2 (Node.js Backend)                       │
│                                                                  │
└── Availability Zone 2                                           │
     ├── Public Subnet 2                                          │
     │     ├── Application Load Balancer                          │
     │     └── NAT Gateway                                        │
     └── Private Subnet 2 (Auto Scaling Group)                    │
           └── Amazon EC2 (Node.js Backend)                       │
                                                                   │
EC2 instances (both AZs) connect out to:  ◄─────────────────────┘
   ├── Amazon SNS  → Amazon SQS → AWS Lambda (Assignment Worker) → Amazon SES
   ├── Amazon S3 (Originals) ⇄ AWS Lambda (Image Resizer) ⇄ Amazon S3 (Resized)
   ├── Amazon DynamoDB
   ├── Amazon Cognito
   └── Amazon CloudWatch (metrics/logs)

Amazon EventBridge (9:00 AM daily rule) → AWS Lambda (Daily Digest) → SNS → SES

Cross-cutting: IAM Roles (permissions for EC2/Lambda), Amazon CloudWatch (observability)
```

### Component Reference

| Component | Type | Role in Architecture |
|---|---|---|
| **End User** | Client | Browser accessing the FlowBoard web app |
| **Amazon CloudFront** | CDN | Distributes the React frontend/static assets; entry point for user traffic |
| **Custom VPC** | Networking | Isolated network hosting all compute resources |
| **Availability Zone 1 & 2** | Networking | Two AZs provide redundancy/high availability |
| **Public Subnet 1 & 2** | Networking | Host the ALB and NAT Gateway per AZ |
| **Private Subnet 1 & 2** | Networking | Host EC2 backend instances, isolated from direct internet access |
| **Application Load Balancer (x2)** | Networking | Distributes incoming API traffic across EC2 instances in each AZ |
| **NAT Gateway (x2)** | Networking | Allows private-subnet EC2 instances outbound internet access (e.g., for package installs, external calls) without inbound exposure |
| **Auto Scaling Group** | Compute | Scales EC2 Node.js backend instances based on load |
| **Amazon EC2 (Node.js Backend)** | Compute | Runs the Express API (per AZ, for redundancy) |
| **Amazon DynamoDB** | Data | Primary NoSQL data store (users, teams, projects, tasks, comments, audit logs) |
| **Amazon S3 (Originals)** | Storage | Stores original uploaded task images |
| **Amazon S3 (Resized)** | Storage | Stores Lambda-generated image thumbnails |
| **AWS Lambda (Image Resizer)** | Compute (serverless) | Triggered by S3 upload; resizes images to thumbnails |
| **Amazon SNS** | Messaging | Publishes task-assignment and daily-digest events |
| **Amazon SQS** | Messaging | Buffers assignment events between SNS and the worker Lambda |
| **AWS Lambda (Assignment Worker)** | Compute (serverless) | Consumes SQS messages, triggers email notifications |
| **Amazon SES** | Messaging | Sends transactional/notification emails |
| **Amazon EventBridge** | Scheduling | Triggers the Daily Digest Lambda on a fixed daily schedule (9:00 AM) |
| **AWS Lambda (Daily Digest)** | Compute (serverless) | Compiles and sends a daily summary via SNS/SES |
| **Amazon Cognito** | Identity | Production authentication/authorization (JWT via Cognito User Pool) |
| **Amazon CloudWatch** | Observability | Centralized metrics and logs for EC2, Lambda, and other services |
| **IAM Roles** | Security | Least-privilege permissions granted to EC2 and Lambda for AWS service access |

---

## 3. Project Structure

```
mini-jira/
├── client/                    # React frontend (Vite)
│   └── src/
│       ├── components/
│       │   ├── kanban/        # KanbanBoard, KanbanColumn, TaskCard
│       │   ├── tasks/         # TaskForm, TaskDetail (with comments)
│       │   ├── layout/        # Sidebar, Header, AppShell
│       │   └── ui/            # Avatar, Badge, Modal, Select, Spinner, EmptyState
│       ├── context/           # AuthContext (JWT + Cognito)
│       ├── lib/               # api.js (Axios), utils.js
│       └── pages/             # Dashboard, Tasks, Projects, Teams, Analytics, Login, Register
│
├── server/                    # Express backend
│   ├── src/
│   │   ├── index.js           # Entry point
│   │   ├── lib/
│   │   │   ├── dynamo.js      # DynamoDB client
│   │   │   ├── s3.js          # S3 client + multer-s3 upload
│   │   │   └── events.js      # SNS publish, CloudWatch metrics
│   │   ├── middleware/
│   │   │   └── auth.js        # JWT / Cognito verify, requireRole
│   │   ├── routes/
│   │   │   ├── auth.js        # POST /auth/register, /auth/login
│   │   │   ├── tasks.js       # Full CRUD + comments + audit
│   │   │   ├── teams.js       # Team management
│   │   │   ├── projects.js    # Project management
│   │   │   └── users.js       # User listing (with team isolation)
│   │   └── lambdas/
│   │       ├── imageResizer.js     # S3 trigger → sharp resize → resized bucket
│   │       └── assignmentWorker.js # SQS trigger → SES email notification
│   └── scripts/
│       ├── createTables.js    # One-time DynamoDB table setup
│       └── seed.js            # Demo data seeder
│
└── .env.example               # Environment variable template
```

---

## 4. AWS Infrastructure Setup

### 4.1 DynamoDB Tables

Run `node scripts/createTables.js` to create 6 tables (all **PAY_PER_REQUEST** billing):

| Table | Global Secondary Indexes |
|---|---|
| `mj-users` | `email-index`, `teamId-index` |
| `mj-teams` | — |
| `mj-projects` | — |
| `mj-tasks` | `teamId-index` (status sort key) |
| `mj-comments` | `taskId-index` |
| `mj-audit` | `taskId-index` |

### 4.2 S3 Buckets

Two buckets are required:

- **`mj-task-images`** — original uploads (private, CORS enabled)
- **`mj-task-images-resized`** — Lambda-generated thumbnails

CORS configuration for the originals bucket:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST"],
    "AllowedOrigins": ["*"],
    "ExposeHeaders": []
  }
]
```

### 4.3 Lambda Functions

Deployed separately (SAM, CDK, or Serverless Framework):

| Function | Trigger | Handler File |
|---|---|---|
| `mj-image-resizer` | S3 `ObjectCreated` on `mj-task-images` | `lambdas/imageResizer.js` |
| `mj-assignment-worker` | SQS `mj-assignment-queue` | `lambdas/assignmentWorker.js` |
| `mj-daily-digest` | EventBridge rule at 9:00 AM daily | `lambdas/dailyDigest.js` |

> The `imageResizer` function requires a **sharp** Lambda layer.

### 4.4 SNS + SQS

- SNS topic: `mj-task-assignments`
- SNS topic: `mj-daily-digest`
- SQS queue: `mj-assignment-queue`
- SQS is subscribed to SNS with **raw message delivery OFF** (the Lambda parses the SNS envelope)
- An SES email endpoint may optionally subscribe directly to SNS for fan-out

### 4.5 Cognito (Production Auth)

1. Create a User Pool with custom attributes: `custom:role`, `custom:teamId`
2. Set `COGNITO_USER_POOL_ID` and `COGNITO_CLIENT_ID` in `.env`
3. The server automatically switches from local JWT to Cognito RS256 verification when configured
4. `POST /api/auth/register` and `POST /api/auth/login` use Cognito when configured, falling back to local auth otherwise

### 4.6 CloudFront (Optional)

Point CloudFront at the S3 originals and resized buckets, and set `CLOUDFRONT_DOMAIN` in `.env` for fast image delivery.

---

## 5. Event-Driven Flows

### 5.1 Task Assignment Flow

```
Manager creates/updates task with assignee
  → Express publishes to SNS topic (mj-task-assignments)
    → SQS queue (mj-assignment-queue) receives message
      → Lambda (assignmentWorker) sends SES email to assignee
```

### 5.2 Image Processing Flow

```
Manager uploads task image
  → multer-s3 uploads to S3 originals bucket (mj-task-images)
    → S3 triggers Lambda (imageResizer)
      → Lambda resizes image to 300px wide
        → Saves thumbnail to S3 resized bucket (mj-task-images-resized)
```

### 5.3 Daily Digest Flow

```
EventBridge rule (9:00 AM daily)
  → Lambda (dailyDigest)
    → Publishes to SNS topic (mj-daily-digest)
      → SES sends digest email
```

> **Note:** All three flows are **fire-and-forget** from the web server — failures in Lambda processing do not affect API response times or success.

---

## 6. Role-Based Access Control (RBAC)

| Feature | Employee | Manager | Admin |
|---|:---:|:---:|:---:|
| View own team's tasks | ✅ | ✅ | ✅ |
| View all teams' tasks | ❌ | ✅ | ✅ |
| Create / edit / delete tasks | ❌ | ✅ | ✅ |
| Update task status | ✅ | ✅ | ✅ |
| Add comments | ✅ | ✅ | ✅ |
| Manage teams | ❌ | ✅ | ✅ |
| Manage projects | ❌ | ✅ | ✅ |
| View analytics | ❌ | ✅ | ✅ |

**Team isolation is enforced server-side** in every route — employees can only access their own team's data regardless of what the client sends. This logic lives in `server/src/middleware/auth.js` (`requireRole`) and is re-checked in each route handler.

---

## 7. API Reference

### 7.1 Auth

| Method | Path | Body |
|---|---|---|
| POST | `/api/auth/register` | `{ name, email, password, role, teamId }` |
| POST | `/api/auth/login` | `{ email, password }` → returns `{ user: { ...fields, token } }` |

### 7.2 Tasks

| Method | Path | Auth |
|---|---|---|
| GET | `/api/tasks` | All roles |
| GET | `/api/tasks/:id` | All roles |
| POST | `/api/tasks` | Manager |
| PATCH | `/api/tasks/:id` | Manager (full) / Employee (status only) |
| DELETE | `/api/tasks/:id` | Manager |
| GET | `/api/tasks/:id/comments` | All roles |
| POST | `/api/tasks/:id/comments` | All roles |
| GET | `/api/tasks/:id/audit` | All roles |

### 7.3 Teams, Projects, Users

Standard CRUD endpoints — see the corresponding route files (`teams.js`, `projects.js`, `users.js`) for full details.

---

## 8. Local Development Setup

### 8.1 Prerequisites

- Node.js 18+
- AWS account with credentials configured (or DynamoDB Local)
- An S3 bucket created (or skip image uploads in dev)

### 8.2 Install dependencies

```bash
cd mini-jira
npm install          # root (concurrently)
cd client && npm install
cd ../server && npm install
```

### 8.3 Configure environment

```bash
cp .env.example server/.env
# Edit server/.env — at minimum set JWT_SECRET and AWS credentials
```

### 8.4 Create DynamoDB tables

```bash
cd server
node scripts/createTables.js

# For DynamoDB Local:
# DYNAMODB_ENDPOINT=http://localhost:8000 node scripts/createTables.js
```

### 8.5 Seed demo data (optional)

```bash
node scripts/seed.js
# Creates demo users, teams, projects, and tasks
```

### 8.6 Start dev servers

```bash
# From the root:
npm run dev
# Client → http://localhost:3000
# Server → http://localhost:5001
```

---

## 9. Design Notes & Rationale

- **Multi-AZ redundancy:** Identical ALB + NAT Gateway + Auto Scaling Group + EC2 setups exist in both AZ1 and AZ2, so the loss of one AZ does not take down the API.
- **Public/private subnet split:** ALBs and NAT Gateways sit in public subnets to handle inbound/outbound internet traffic; EC2 application instances sit in private subnets, reachable only through the ALB.
- **Asynchronous side effects:** Notification emails and image resizing are offloaded to SNS/SQS/Lambda so they never block or fail a user-facing API request.
- **Central identity:** Cognito is the production identity provider, replacing local JWT auth once configured, while keeping local auth as a fallback for development.
- **Observability & governance:** CloudWatch aggregates logs/metrics from EC2 and Lambda; IAM roles scope permissions per service to enforce least privilege.

---

## 10. Reference Materials

- Architecture diagram: *Horizontal AWS HA Architecture* (see Section 2)
- Demo video: available via the project's shared drive link
- Demo accounts: seeded by `scripts/seed.js` for Manager and Employee roles (see `.env.example` / seed script for current credentials — avoid committing real credentials to version control or public documentation)