# Opti Advisr

> **Serverless AWS Cost Optimizer** with AI-powered savings advice.

An interactive dashboard that helps teams visualise AWS spend, manage cloud resources, and get AI-driven cost-optimisation recommendations — all running on a fully serverless stack.

---

## Architecture

```
Frontend (React + Vite)          →  Cloudflare Pages (CDN)
       ↕ HTTPS / x-api-key
AWS API Gateway (REST, Regional) →  ap-south-1
       ↕ AWS_PROXY
┌──────────────────────────────────────────────────┐
│  Lambda: cost-retriever       → Cost Explorer    │
│  Lambda: resource-manager     → EC2, RDS, S3     │
│  Lambda: ai-agent (LangChain) → S3 (Agent Data)  │
└──────────────────────────────────────────────────┘
       ↕
S3 Buckets  (opti-advisr-app-data-*, opti-advisr-agent-data-*)
```

| Layer | Technology |
|-------|-----------|
| Frontend | React 19, Vite 6, Recharts, Lucide Icons, Sonner |
| Hosting | Cloudflare Pages / Workers |
| API | AWS API Gateway (REST, API Key auth) |
| Compute | AWS Lambda (Python 3.12) |
| Data | AWS Cost Explorer, EC2, RDS, S3 |
| IaC | Terraform ≥ 1.5 |

---

## Prerequisites

| Tool | Version |
|------|---------|
| Node.js | ≥ 20 |
| npm | ≥ 10 |
| Python | ≥ 3.12 |
| AWS CLI | ≥ 2.x (configured with `aws configure`) |
| Terraform | ≥ 1.5 |
| Wrangler (optional) | ≥ 4.x (for Cloudflare deployment) |

---

## Quick Start

### 1. Clone & Install

```bash
git clone https://github.com/Opti-Advisr/Opti-Advisr.git
cd Opti-Advisr
```

### 2. Deploy Backend (AWS)

```bash
# Package Lambda functions
cd backend/lambdas
zip cost_retriever.zip -j cost_retriever/handler.py
zip resource_manager.zip -j resource_manager/handler.py
zip ai_agent.zip -j ai_agent/handler.py
cd ../../

# Deploy infrastructure
cd infra
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

After `terraform apply`, note the outputs:

| Output | Description |
|--------|-------------|
| `api_base_url` | Your API Gateway endpoint |
| `api_key_id` | API Key ID (retrieve the value via AWS Console or CLI) |

Retrieve your API key value:

```bash
aws apigateway get-api-key --api-key <api_key_id> --include-value --query 'value' --output text
```

### 3. Configure Frontend

```bash
cd frontend
cp .env.example .env.local
```

Edit `.env.local` with your actual values:

```env
# For Vite dev server proxy
API_BASE_URL=https://<your-api-id>.execute-api.ap-south-1.amazonaws.com/v1
API_KEY=<your-api-key>

# For production builds (baked into the bundle)
VITE_API_URL=https://<your-api-id>.execute-api.ap-south-1.amazonaws.com/v1
VITE_API_KEY=<your-api-key>
```

### 4. Run Locally

```bash
cd frontend
npm install
npm run dev
```

Open [http://localhost:5173](http://localhost:5173).

The Vite dev server proxies `/api` requests to your API Gateway, injecting the API key automatically.

### 5. Deploy Frontend to Cloudflare

```bash
cd frontend
npm run build
npx wrangler deploy
```

---

## Project Structure

```
Opti-Advisr/
├── backend/
│   └── lambdas/
│       ├── cost_retriever/
│       │   └── handler.py
│       ├── resource_manager/
│       │   └── handler.py
│       ├── ai_agent/
│       │   └── handler.py
│       ├── cost_retriever.zip
│       ├── resource_manager.zip
│       └── ai_agent.zip
├── frontend/
│   ├── src/
│   │   ├── components/ui/
│   │   ├── lib/api.js
│   │   ├── pages/Home.jsx
│   │   ├── utils/demoData.js
│   │   ├── styles/index.css
│   │   ├── App.jsx
│   │   └── main.jsx
│   ├── .env.example
│   ├── index.html
│   ├── package.json
│   ├── vite.config.js
│   └── wrangler.jsonc
├── infra/
│   ├── main.tf
│   ├── modules/
│   ├── cost_retriever.tf
│   ├── resource_manager.tf
│   └── ai_agent.tf
├── lambdas/
├── layers/
├── .github/
│   ├── workflows/ci.yml
│   └── pull_request_template.md
├── .gitignore
├── CONTRIBUTING.md
├── LICENSE
└── README.md
```

---

## API Endpoints

All endpoints require the `x-api-key` header.

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/costs` | Retrieve cost data from AWS Cost Explorer |
| `GET` | `/resources?type=all` | List EC2 instances, RDS databases, S3 buckets |
| `POST` | `/resources` | Perform actions on resources (stop, start, terminate) |
| `POST` | `/agent` | Send a prompt to the AI cost advisor |

Example:

```bash
curl -X POST https://<api-id>.execute-api.ap-south-1.amazonaws.com/v1/costs \
  -H "x-api-key: <your-key>" \
  -H "Content-Type: application/json" \
  -d '{"period": "monthly"}'
```

---

## Environment Variables

### Frontend (`.env.local`)

| Variable | Used By | Description |
|----------|---------|-------------|
| `API_BASE_URL` | Vite dev proxy | Target URL for the dev server proxy |
| `API_KEY` | Vite dev proxy | API key injected by the proxy |
| `VITE_API_URL` | Production build | API Gateway URL baked into the JS bundle |
| `VITE_API_KEY` | Production build | API key baked into the JS bundle |

### Lambda Environment (set via Terraform)

| Variable | Description |
|----------|-------------|
| `PROJECT_NAME` | `opti-advisr` |
| `APP_DATA_BUCKET` | S3 bucket for app state |
| `AGENT_DATA_BUCKET` | S3 bucket for AI agent data |
| `RESOURCE_TAG_KEY` | Tag key for managed resources |
| `RESOURCE_TAG_VALUE` | Tag value for managed resources |

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for branch workflow, commit conventions, and pull request guidelines.

---

## License

[MIT](LICENSE)
