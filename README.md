<div align="center">

# ⚡ Asynchronous Distributed Report Processing System

**Production-grade asynchronous job processing platform built with FastAPI, React 18, AWS SQS, DynamoDB, and resilient Python workers.**

[![Python](https://img.shields.io/badge/Python-3.11-blue.svg?logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.100+-009688.svg?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![React](https://img.shields.io/badge/React-18-61DAFB.svg?logo=react&logoColor=black)](https://react.dev)
[![AWS SQS](https://img.shields.io/badge/AWS-SQS-FF9900.svg?logo=amazonsqs&logoColor=white)](https://aws.amazon.com/sqs/)
[![DynamoDB](https://img.shields.io/badge/AWS-DynamoDB-4053D6.svg?logo=amazondynamodb&logoColor=white)](https://aws.amazon.com/dynamodb/)
[![Docker](https://img.shields.io/badge/Docker-Compose%20v2-2496ED.svg?logo=docker&logoColor=white)](https://docker.com)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC.svg?logo=terraform&logoColor=white)](https://terraform.io)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

[English](README.md) | [Español](README.es.md)

</div>

---

## 📖 Overview

This repository provides an enterprise-ready, distributed, asynchronous report processing platform for analytics and SaaS applications. 

Clients submit on-demand report requests via a secure REST API. Requests are decoupled via message queues (**AWS SQS**) with multi-tier priority handling, processed concurrently by resilient background workers (**Python `asyncio`** with circuit breakers and retries), and persisted in **AWS DynamoDB**. Real-time job updates are streamed to the frontend over **Server-Sent Events (SSE)**.

---

## 🏛️ System Architecture

```mermaid
flowchart TD
    subgraph ClientLayer ["Client Layer"]
        UI["React 18 SPA (TypeScript + Vite)"]
    end

    subgraph APILayer ["API Gateway & Core API"]
        API["FastAPI Backend (Python 3.11)"]
        Auth["JWT Authentication & RBAC"]
        SSE["SSE Stream Service (/stream/jobs)"]
    end

    subgraph MessagingLayer ["Messaging & Queuing (AWS SQS)"]
        HighQ[("High-Priority SQS Queue")]
        StdQ[("Standard SQS Queue")]
        DLQ[("Dead Letter Queue (DLQ)")]
    end

    subgraph WorkerLayer ["Distributed Workers (asyncio)"]
        Worker["Concurrent Queue Consumer"]
        CB["Circuit Breaker & Exponential Backoff"]
        Processor["Report Generator Engine"]
    end

    subgraph StorageLayer ["Data Persistence (AWS DynamoDB)"]
        JobsTable[("Jobs State Table")]
        UsersTable[("Users Auth Table")]
    end

    UI -->|1. Submit Job Request (JWT)| API
    API --> Auth
    API -->|2. Write Initial State (PENDING)| JobsTable
    API -->|3. Publish Message| HighQ
    API -->|3. Publish Message| StdQ
    
    Worker -->|4. Poll & Consume Messages| HighQ
    Worker -->|4. Poll & Consume Messages| StdQ
    Worker -.->|Excessive Failures| DLQ
    Worker --> CB --> Processor
    Processor -->|5. Update Progress & Results| JobsTable
    
    JobsTable -.->|State Mutation Stream| SSE
    SSE -.->|6. Push Live Status via SSE| UI
```

---

## ✨ Key Architectural Features

- **⚡ Asynchronous Decoupling:** API requests return immediately with an HTTP 202 Accepted status and job tracking ID, preventing timeout bottlenecks on heavy analytical tasks.
- **🚦 Dual Priority Queues:** Dedicated SQS queues for standard vs. high-priority jobs to ensure SLA compliance for critical requests.
- **🔄 Fault-Tolerant Worker Pool:**
  - Implements **Circuit Breakers** and **Exponential Backoff with Full Jitter** to prevent cascading downstream outages.
  - Automatic Dead-Letter Queue (DLQ) routing for poisoned or repeatedly failing payloads.
- **📡 Real-Time Server-Sent Events (SSE):** Frontends stream live progress updates (`PENDING` $\rightarrow$ `PROCESSING` $\rightarrow$ `COMPLETED` / `FAILED`) without polling spam.
- **💾 Idempotent DynamoDB State:** All state transitions and result payloads are stored with optimistic locking and atomic status updates.
- **☁️ LocalStack Zero-Cost Emulation:** Full offline local development environment emulating AWS SQS and DynamoDB via LocalStack.
- **🏗️ Infrastructure as Code (Terraform):** Production-ready Terraform configurations for automated provisioning of AWS cloud infrastructure.
- **🧪 Rigorous Verification Suite:** Comprehensive test coverage using `pytest`, `pytest-asyncio`, AWS `moto` mocking, and property-based testing with `hypothesis`.

---

## 🚀 Quick Start (Local Development)

### Prerequisites
- [Docker](https://www.docker.com/) & Docker Compose v2+
- Git
- *(Optional)* Python 3.11+ and Node.js 18+ for local non-containerized debugging.

### 1. Clone & Configure Environment
```bash
git clone https://github.com/JuanCeron023/asynchronous-Report-Processing-SystemV1.git
cd asynchronous-Report-Processing-SystemV1

# Copy environment variables
cp .env.example .env
```

### 2. Launch Stack via Docker Compose
```bash
docker compose up -d --build
```

This single command provisions the complete ecosystem:
| Service | URL / Port | Purpose |
|---|---|---|
| **Frontend Web UI** | [http://localhost:3000](http://localhost:3000) | React 18 dashboard with real-time SSE job monitoring |
| **Backend API** | [http://localhost:8000](http://localhost:8000) | FastAPI core REST API |
| **OpenAPI Docs (Swagger)** | [http://localhost:8000/docs](http://localhost:8000/docs) | Interactive API exploration and testing |
| **LocalStack AWS** | [http://localhost:4566](http://localhost:4566) | Local AWS SQS & DynamoDB emulator |
| **Background Worker** | *Internal container* | Autonomous asyncio SQS consumer & report generator |

### 3. Verify Health
```bash
curl -s http://localhost:8000/health
# {"status":"healthy","services":{"dynamodb":"up","sqs":"up"}}
```

---

## 📡 API Specification

All protected endpoints require a valid Bearer token in the `Authorization` header (`Bearer <JWT>`).

| Method | Endpoint | Description | Auth Required |
|---|---|---|:---:|
| `POST` | `/auth/register` | Register a new user | No |
| `POST` | `/auth/login` | Authenticate and obtain JWT token | No |
| `POST` | `/jobs` | Create a new report processing job | **Yes** |
| `GET` | `/jobs` | List user's jobs with pagination and filters | **Yes** |
| `GET` | `/jobs/{job_id}` | Query current status and results of a job | **Yes** |
| `GET` | `/stream/jobs` | Server-Sent Events (SSE) live status stream | **Yes** |
| `GET` | `/health` | Service health check & AWS connectivity | No |

### Example: Creating a Report Job

```bash
# 1. Login to retrieve token
TOKEN=$(curl -s -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"secretpassword"}' | jq -r .access_token)

# 2. Submit high-priority report request
curl -s -X POST http://localhost:8000/jobs \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "report_type": "sales_summary",
    "priority": "HIGH",
    "parameters": {
      "start_date": "2026-01-01",
      "end_date": "2026-09-01"
    }
  }'
```

---

## 💻 Non-Docker Development

### Backend API
```bash
cd backend
python -m venv .venv
source .venv/bin/activate # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

### Worker Process
```bash
cd worker
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m app.main
```

### Frontend UI
```bash
cd frontend
npm install
npm run dev
```

### Running Test Suite
```bash
cd backend
pip install pytest pytest-asyncio moto[all] httpx hypothesis
pytest tests/ -v
```

---

## 🏗️ Production Cloud Deployment (Terraform)

Infrastructure can be provisioned directly to AWS using Terraform:

```bash
cd infra/terraform
terraform init
terraform plan
terraform apply -auto-approve
```

See [`DEPLOYMENT_GUIDE.md`](DEPLOYMENT_GUIDE.md) and [`TECHNICAL_DOCS.md`](TECHNICAL_DOCS.md) for architecture details, IAM policies, and VPC configuration.

---

## 📁 Repository Structure

```
├── backend/                  # FastAPI REST API application
│   ├── app/
│   │   ├── auth/             # JWT authentication & security dependencies
│   │   ├── db/               # DynamoDB repository & data models
│   │   ├── jobs/             # Job creation, listing & lifecycle handlers
│   │   ├── queue/            # SQS message publishing client
│   │   ├── stream/           # Server-Sent Events (SSE) streaming
│   │   └── observability/    # Structured JSON logging & Prometheus metrics
├── worker/                   # Distributed background consumer
│   ├── app/
│   │   ├── consumer.py       # SQS long-polling consumer
│   │   ├── processor.py      # Report calculation & state transition engine
│   │   ├── circuit_breaker.py# Resilience circuit breaker pattern
│   │   └── retry.py          # Backoff and retry policies
├── frontend/                 # React 18 + Vite + TypeScript dashboard
│   ├── src/
│   │   ├── components/       # JobForm, JobList, StatusBadges, Toasts
│   │   ├── hooks/            # useAuth, useJobs, useSSE (real-time stream)
│   │   └── pages/            # Dashboard and Authentication views
├── infra/
│   ├── terraform/            # AWS production infrastructure as code
│   └── localstack/           # Local AWS SQS & DynamoDB initialization scripts
├── docker-compose.yml        # Multi-service local orchestrator
├── DEPLOYMENT_GUIDE.md       # Step-by-step production deployment manual
├── TECHNICAL_DOCS.md         # In-depth architectural specifications
└── LICENSE                   # MIT License
```

---

## 📄 License

This project is licensed under the [MIT License](LICENSE). Developed by [Juan Cerón](https://github.com/JuanCeron023).
