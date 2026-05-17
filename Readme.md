# Book Shop - Phase 2 CI/CD Pipelines

**Mohammad Awad - 20210577**
**Laith Alzubi - 20231224**

Group of 2

---

## About

This is Phase 2 of the book shop project. We added three CI/CD pipelines using GitHub Actions. Each pipeline runs on a different branch and deploys to the same EC2 instance but on different ports so they don't conflict with each other.

We made a separate repo for Phase 2 to keep our Phase 1 repo untouched.

---

## The Three Branches

### dev - Artifact First

Triggers on every push to dev.

What it does:
1. Installs dependencies and packages the source code into a .tar.gz file (named with the commit sha so each build gets its own file)
2. Commits that file to the artifacts/ folder in the repo - this is the audit trail
3. Builds the Docker image from the artifact (the Dockerfile extracts the archive, it does NOT use the source directly)
4. Pushes the image to Docker Hub tagged as `dev-<sha>`
5. SSHs into EC2 and runs the containers

Port: **8001**

The reason we build from the artifact and not from source is because this is the artifact-first philosophy - what gets packaged is exactly what gets deployed.

### test - Image First

Triggers on push to test (we merge from dev).

What it does:
1. Builds a fresh artifact from source (does not reuse the one from dev)
2. Builds the Docker image from it
3. Pushes to Docker Hub tagged as `<sha>` (no prefix, since this is the version that will go to prod)
4. SSHs into EC2 and deploys by pulling from Docker Hub

Port: **8002**

We are a group of 2 so we push to Docker Hub (not ECR).

### prod - Promotion Only

Triggers on push/merge to prod.

This pipeline never builds anything. It reads the IMAGE_VERSION variable from GitHub repository settings (Settings → Secrets and variables → Actions → Variables) and pulls that exact image from Docker Hub to deploy.

To release a new version to prod we update IMAGE_VERSION to the sha we want, then push to prod.

Port: **8000**

---

## How They Coexist on One EC2

All three run on the same EC2 without conflicts because:

- Different compose project names: `-p dev`, `-p test`, `-p prod`
- Different host ports: 8001, 8002, 8000
- Separate postgres volumes for each: pgdata_dev, pgdata_test, pgdata_prod
- Each environment writes its own .env file to a separate folder (/app/dev, /app/test, /app/prod)

---

## Secrets and Variables

Variables (not sensitive):
- `EC2_HOST` - the IP of the EC2 instance
- `IMAGE_VERSION` - the image tag to pull in prod

Secrets:
- `EC2_SSH_KEY` - private key to SSH into the server
- `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` - for pushing/pulling images
- `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` - database config
- `SECRET_KEY` - Django secret key

---

## Group Size and Matching Choices

We are a group of 2.

Because of this we used Docker Hub as our registry instead of AWS ECR. The test pipeline pushes the image to Docker Hub and the prod pipeline pulls from there.

Also since we are a group of 2, we run docker-compose commands directly on the EC2 server through SSH. We did not need to automate updating the docker-compose.yml file inside the pipeline (that is only needed for groups of 3).

---

## Docker Hub

Images are at `mhmdawad/book-shop`

- dev pipeline pushes: `mhmdawad/book-shop:dev-<sha>`
- test pipeline pushes: `mhmdawad/book-shop:<sha>`
- prod pulls whatever tag is in IMAGE_VERSION (which will be one of the test tags)
