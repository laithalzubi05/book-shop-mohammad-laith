Mohammad Awad - 20210577
Laith Alzu'bi - 20231224
# book-shop-mohammad-laith

This is Phase 2 of the book shop project. We set up three pipelines using GitHub Actions. Each one runs on its own branch and deploys to EC2.

### dev

Every time we push to dev, the pipeline:
1. Packages the code into a zip file (tar.gz) and saves it in the artifacts folder
2. Builds a Docker image from that zip file
3. Pushes the image to Docker Hub
4. Deploys it to EC2 on port 8001

We save the zip file in the repo so there is a history of every build.

### test

Every time we push to test, the pipeline:
1. Packages the code again from scratch (does not reuse the dev zip)
2. Builds a Docker image from it
3. Pushes it to Docker Hub
4. Deploys it to EC2 on port 8002

### prod

Every time we push to prod, the pipeline:
1. Reads the IMAGE_VERSION variable from GitHub settings
2. Pulls that image from Docker Hub
3. Deploys it to EC2 on port 8000

---------------------------------------------------------------------------------------------------------------

## How all three run on the same server

We use different ports so they don't conflict:
- dev runs on port 8001
- test runs on port 8002
- prod runs on port 8000

We also use different compose project names (-p dev, -p test, -p prod) and separate database volumes so each environment has its own data.

---------------------------------------------------------------------------------------------------------------

## Group size choices

Since we are a group of 2:
- We use Docker Hub instead of AWS ECR
- We run docker-compose directly on the server over SSH (groups of 3 need to automate this inside the pipeline)

---------------------------------------------------------------------------------------------------------------

## Secrets and variables we set up in GitHub

- EC2_HOST - the server IP
- IMAGE_VERSION - the image version to deploy to prod
- EC2_SSH_KEY - the key to connect to the server
- DOCKERHUB_USERNAME and DOCKERHUB_TOKEN - to push and pull images
- POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD - database settings
- SECRET_KEY - Django secret key
