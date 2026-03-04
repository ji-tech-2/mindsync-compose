# MindSync — Docker Compose Setup

This repository contains the Docker Compose configuration to run the entire MindSync platform locally with a single command.

## Architecture Overview

| Service | Image / Build | Port | Description |
|---|---|---|---|
| **frontend** | `ivanchristian005/mindsync-frontend:latest` | `3000` | React frontend (served via Nginx) |
| **api-gateway** | `clauhan/kong-gateway:latest` | `8000` (proxy) / `8001` (admin) | Kong API Gateway — routes traffic to backend & inference |
| **backend** | `joshmoritty/mindsync-backend:latest` | `8080` | Spring Boot auth & core API |
| **inference** | `thomasseow/mindsync-model-flask:latest` | `5000` | Flask ML inference service |
| **database** | `postgres:16-alpine` | `5432` | PostgreSQL for backend (Spring Boot) |
| **database-flask** | `postgres:16-alpine` | `5433` | PostgreSQL for inference (Flask) |
| **cache** | `valkey/valkey:7-alpine` | `6379` | Valkey (Redis-compatible) cache |
| **training** | Built from `./mindsync-model-training` | — | Model training job (on-demand, via profile) |

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) (v20.10+)
- [Docker Compose](https://docs.docker.com/compose/install/) (v2.x — included with Docker Desktop)
- The following API keys / credentials (see [Environment Variables](#environment-variables))

## Quick Start

### 1. Clone the repository

```bash
git clone <repo-url> mindsync-compose
cd mindsync-compose
```

### 2. Obtain Required API Keys & Credentials

Before creating the `.env` file, you need to sign up and get API keys from the following services:

#### Weights & Biases (W&B) — required for model training & inference

1. Go to [https://wandb.ai/site](https://wandb.ai/site) and create a free account (or log in).
2. After logging in, go to **Settings** → **API Keys** ([https://wandb.ai/authorize](https://wandb.ai/authorize)).
3. Copy your **API key** — this is your `WANDB_API_KEY`.

#### Google Gemini API — required for AI features

1. Go to [https://aistudio.google.com/apikey](https://aistudio.google.com/apikey) and sign in with your Google account.
2. Click **Create API Key** and select a Google Cloud project (or create one).
3. Copy the generated key — this is your `GEMINI_API_KEYS`.

#### Resend — for sending emails (optional, alternative to Gmail SMTP)

1. Go to [https://resend.com](https://resend.com) and create a free account.
2. Navigate to **API Keys** in the dashboard and create a new key.
3. Copy the key — this is your `RESEND_API`.

#### Gmail SMTP — for sending OTP emails (optional, alternative to Resend)

1. Sign in to your Google Account at [https://myaccount.google.com](https://myaccount.google.com).
2. Go to **Security** → **2-Step Verification** (must be enabled).
3. Under 2-Step Verification, scroll down to **App passwords** and generate a new one (select app: *Mail*).
4. Copy the 16-character app password — this is your `SMTP_PASSWORD`.
5. Your Gmail address is your `SMTP_USERNAME` and `SMTP_FROM_EMAIL`.

> **Generating JWT RSA keys (if you don't have them yet):**
>
> ```bash
> # Generate private key
> openssl genpkey -algorithm RSA -out private_key.pem -pkeyopt rsa_keygen_bits:2048
>
> # Extract public key
> openssl rsa -pubout -in private_key.pem -out public_key.pem
> ```
>
> Then paste the contents of `private_key.pem` and `public_key.pem` into the `.env` file.

### 3. Create the `.env` file

Copy the example environment file and fill in your actual values:

```bash
cp .env.example .env
```

Then open `.env` and replace the placeholder values:

```dotenv
# Do not Change!!
WANDB_ENTITY=thomas-seowinata-binus-university
WANDB_PROJECT=mindsync-model

WANDB_API_KEY=your_wandb_api_key
GEMINI_API_KEYS=your_gemini_api_key

# Gmail SMTP (for sending OTP emails)
SMTP_USERNAME=your_smtp_username
SMTP_PASSWORD=your_smtp_password
SMTP_FROM_EMAIL=your_from_email
RESEND_API=your_resend_api_key

# JWT RSA Keys (shared between backend & inference)
JWT_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----
<your private key content>
-----END PRIVATE KEY-----"

JWT_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----
<your public key content>
-----END PUBLIC KEY-----"
```


### 4. Clone the API Gateway config

The Kong API Gateway requires a declarative config file. Clone the gateway repository inside this project directory:

```bash
git clone https://github.com/ji-tech-2/mindsync-api-gateway.git
```

This will create a `mindsync-api-gateway/` folder containing `kong.local.yml`, which is mounted into the Kong container.

### 5. Start all services

```bash
docker compose up -d
```

Docker will pull all required images and start the services. On first run this may take a few minutes.

### 6. Verify services are running

```bash
docker compose ps
```

All services should show a status of `running` (databases and cache should show `healthy`).

### 7. Access the application

| Endpoint | URL |
|---|---|
| Frontend | [http://localhost:3000](http://localhost:3000) |
| API Gateway (proxy) | [http://localhost:8000](http://localhost:8000) |
| Backend API (direct) | [http://localhost:8080](http://localhost:8080) |
| Inference API (direct) | [http://localhost:5000](http://localhost:5000) |

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `WANDB_API_KEY` | Yes | — | Weights & Biases API key |
| `GEMINI_API_KEYS` | Yes | — | Google Gemini API key |
| `SMTP_USERNAME` | No | — | Gmail SMTP username (for OTP emails) |
| `SMTP_PASSWORD` | No | — | Gmail SMTP app password |
| `SMTP_FROM_EMAIL` | No | — | Sender email address |
| `RESEND_API` | No | — | Resend API key (alternative email provider) |
| `JWT_PRIVATE_KEY` | Yes | — | RSA private key (PEM format) — used by backend |
| `JWT_PUBLIC_KEY` | Yes | — | RSA public key (PEM format) — used by backend & inference |
| `WANDB_PROJECT` | No | `mindsync-model` | W&B project name |
| `ARTIFACT_VERSION` | No | `latest` | Model artifact version to use |
| `MODEL_VERSION` | No | `v1.0` | Model version tag (training) |
| `SPRING_MAIL_HOST` | No | `smtp.gmail.com` | SMTP mail host |
| `SPRING_MAIL_PORT` | No | `587` | SMTP mail port |

## Running the Training Service

The training service uses a Docker Compose **profile** and does not start by default. To run it:

```bash
docker compose --profile training up training
```

Trained artifacts will be saved to `./mindsync-model-training/artifacts/`.

## Common Commands

```bash
# Start all services in detached mode
docker compose up -d

# View logs of all services
docker compose logs -f

# View logs of a specific service
docker compose logs -f backend

# Stop all services
docker compose down

# Stop all services and remove volumes (⚠️ deletes database data)
docker compose down -v

# Rebuild and restart a specific service
docker compose up -d --pull always backend

# Pull latest images
docker compose pull
```

## Troubleshooting

### Services fail to start — health check issues

The backend and inference services wait for their databases and cache to be healthy before starting. If they keep restarting, check the database/cache logs:

```bash
docker compose logs database
docker compose logs database-flask
docker compose logs cache
```

### Port conflicts

If a port is already in use on your machine, either stop the conflicting service or change the host port mapping in `docker-compose.yml`. For example, to change the frontend port:

```yaml
frontend:
  ports:
    - "3001:80"  # changed from 3000 to 3001
```

### Kong gateway errors

Make sure the Kong declarative config file exists at `./mindsync-api-gateway/kong.local.yml`. This file is mounted into the gateway container.

### Resetting everything

To do a full reset (removes all containers, networks, and volumes):

```bash
docker compose down -v
docker compose up -d
```
