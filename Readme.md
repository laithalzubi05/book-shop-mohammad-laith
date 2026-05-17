# Book Shop — CI/CD Pipeline Project

**Mohammad Awad — 20210577**
**Laith Alzubi — 20231224**

---

## Project Overview

A Django + PostgreSQL containerized web application with three automated CI/CD pipelines deployed to AWS EC2. The app is a book shop built with Django 4.0.4.

- **Source repository:** [laithalzubi05/book-shop-mohammad-laith](https://github.com/laithalzubi05/book-shop-mohammad-laith)
- **Docker Hub image:** `mhmdawad/book-shop`
- **Deployment target:** AWS EC2 (Ubuntu 24.04)

---

## Architecture

```
Django App (port 8000 internal)
    └── PostgreSQL 15 (pgdata volume)
```

Two-tier architecture: Django backend + PostgreSQL database, orchestrated with Docker Compose. No reverse proxy (group of 2).

---

## Branch Strategy

| Branch | Pipeline Type | Port | Docker Tag |
|--------|--------------|------|------------|
| `dev`  | Artifact-First | 8001 | `dev-<sha>` |
| `test` | Image-First    | 8002 | `<sha>` |
| `prod` | Promotion-Only | 8000 | `vars.IMAGE_VERSION` |

---

## Pipeline 1 — Dev (Artifact-First)

**Trigger:** push to `dev` branch

**Steps:**
1. Checkout source code
2. Set up Python 3.11
3. Install Python dependencies
4. Sync with remote dev branch (git pull)
5. Package source into `artifacts/app-<sha>.tar.gz` and commit to repo
6. Copy artifact into Docker build context
7. Build Docker image `FROM artifact.tar.gz` — no raw source in image
8. Push image to Docker Hub as `mhmdawad/book-shop:dev-<sha>`
9. SSH to EC2 → write `.env` → deploy with `docker compose -p dev up -d` on port **8001**

**Key concept:** The Docker image is built from a packaged artifact (tar.gz), not directly from source. The artifact is version-controlled in `artifacts/`.

---

## Pipeline 2 — Test (Image-First)

**Trigger:** push to `test` branch

**Steps:**
1. Checkout source code
2. Set up Python 3.11
3. Install Python dependencies
4. Build a fresh artifact from source (not reused from dev)
5. Copy artifact into Docker build context
6. Build Docker image and push to Docker Hub as `mhmdawad/book-shop:<sha>`
7. SSH to EC2 → pull image → deploy with `docker compose -p test up -d` on port **8002**

**Key concept:** The image is built fresh from source (not promoted from dev), then pushed to Docker Hub and pulled on EC2. No artifact is committed to the repo.

---

## Pipeline 3 — Prod (Promotion-Only)

**Trigger:** push to `prod` branch

**Steps:**
1. Read `IMAGE_VERSION` from GitHub repository variable (no build)
2. SSH to EC2 → pull existing image `mhmdawad/book-shop:<IMAGE_VERSION>` → deploy with `docker compose -p prod up -d` on port **8000**

**Key concept:** Zero Docker builds in production. An already-tested image is promoted by setting `vars.IMAGE_VERSION` in GitHub to the tested SHA, then pushing to prod.

---

## Environment Variables (GitHub Secrets & Variables)

### Secrets
| Name | Description |
|------|-------------|
| `DOCKERHUB_USERNAME` | Docker Hub username (`mhmdawad`) |
| `DOCKERHUB_TOKEN` | Docker Hub access token |
| `EC2_SSH_KEY` | Private key (.pem) for EC2 SSH access |
| `POSTGRES_DB` | PostgreSQL database name |
| `POSTGRES_USER` | PostgreSQL username |
| `POSTGRES_PASSWORD` | PostgreSQL password |
| `SECRET_KEY` | Django secret key |

### Variables
| Name | Description |
|------|-------------|
| `EC2_HOST` | EC2 public IP address |
| `IMAGE_VERSION` | Image tag to deploy to prod (set manually before prod push) |

---

## Docker Compose Project Isolation

Each environment runs as a separate Docker Compose project on the same EC2 instance:

```
docker compose -p dev  up -d   # port 8001
docker compose -p test up -d   # port 8002
docker compose -p prod up -d   # port 8000
```

Each project has its own named volume (`pgdata_dev`, `pgdata_test`, `pgdata_prod`) so databases are fully isolated.

---

## How to Trigger Each Pipeline

**Dev:** commit and push to `dev` branch — pipeline runs automatically.

**Test:** merge `dev` into `test` and push:
```bash
git checkout test
git merge dev
git push origin test
```

**Prod:**
1. Go to GitHub → Settings → Variables → set `IMAGE_VERSION` to the tested SHA
2. Merge `test` into `prod` and push:
```bash
git checkout prod
git merge test
git push origin prod
```

---

## Project Structure

```
book-shop-mohammad-laith/
├── .github/
│   └── workflows/
│       ├── dev.yml        # Artifact-First pipeline
│       ├── test.yml       # Image-First pipeline
│       └── prod.yml       # Promotion-Only pipeline
├── book-shop/
│   ├── Dockerfile         # Builds from artifact.tar.gz
│   ├── docker-compose.yml # Uses IMAGE_NAME:IMAGE_TAG env vars
│   ├── requirements.txt
│   └── manage.py
├── artifacts/             # Versioned build artifacts (committed by CI)
├── .gitignore
└── README.md
```
