# Cart CI

This directory contains the RoboShop `cart` service source code and the Jenkins
CI entrypoint. The CI job builds the Node.js service as a Docker image, pushes
the image to AWS ECR, and triggers the matching cart CD job.

## Files

| File | Purpose |
| --- | --- |
| `Jenkinsfile` | Loads `jenkins-shared-library` and calls the shared Node.js CI pipeline. |
| `server.js` | Express API for cart operations. |
| `package.json` | Node.js metadata, dependencies, and version used as the Docker image tag. |
| `Dockerfile` | Builds the cart container image. |

## Runtime Configuration

| Variable | Default | Purpose |
| --- | --- | --- |
| `CART_SERVER_PORT` | `8080` | Port used by the cart service. |
| `REDIS_HOST` | `redis` | Redis hostname for cart storage. |
| `CATALOGUE_HOST` | `catalogue` | Catalogue service hostname. |
| `CATALOGUE_PORT` | `8080` | Catalogue service port. |

## API Endpoints

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `GET` | `/health` | Returns app and Redis connection status. |
| `GET` | `/metrics` | Returns Prometheus metrics. |
| `GET` | `/cart/:id` | Gets one cart by ID. |
| `DELETE` | `/cart/:id` | Deletes one cart by ID. |
| `GET` | `/rename/:from/:to` | Renames a cart ID, usually after login. |
| `GET` | `/add/:id/:sku/:qty` | Adds a product to a cart. |
| `GET` | `/update/:id/:sku/:qty` | Updates item quantity in a cart. |
| `POST` | `/shipping/:id` | Adds or updates shipping cost in a cart. |

## Local Run

Install dependencies:

```bash
npm install
```

Run the service:

```bash
CART_SERVER_PORT=8080 REDIS_HOST=localhost CATALOGUE_HOST=localhost CATALOGUE_PORT=8080 node server.js
```

Check health:

```bash
curl http://localhost:8080/health
```

## Docker

Build locally:

```bash
docker build -t roboshop/cart:local .
```

Run locally:

```bash
docker run --rm -p 8080:8080 \
  -e REDIS_HOST=host.docker.internal \
  -e CATALOGUE_HOST=host.docker.internal \
  -e CATALOGUE_PORT=8080 \
  roboshop/cart:local
```

## Jenkins CI Flow

The `Jenkinsfile` loads the shared library:

```groovy
@Library('jenkins-shared-library') _
```

It passes this service configuration:

```groovy
def configMap = [
    project: "roboshop",
    component: "cart"
]
```

Branch behavior:

| Branch | Behavior |
| --- | --- |
| Any branch except `main` | Calls `nodeJSEKSPipeline(configMap)`. |
| `main` | Stops and prints `Please follow the CR process`. |

The shared CI logic lives in:

```text
../jenkins-shared-library/vars/nodeJSEKSPipeline.groovy
```

Current CI stages:

1. Read `version` from `package.json`.
2. Run `npm install`.
3. Run the current unit-test placeholder `echo test`.
4. Build and push the Docker image to ECR.
5. Trigger the cart CD job with `appVersion` and `deploy_to=dev`.
6. Clean the Jenkins workspace.

## Image Version

The Docker image tag comes from `package.json`:

```json
{
  "version": "1.0.0"
}
```

For this version, CI builds an image like:

```text
704887592073.dkr.ecr.us-east-1.amazonaws.com/roboshop/cart:1.0.0
```

## Jenkins Requirements

The Jenkins CI agent must have label:

```text
AGENT-CI
```

Required tools and credentials:

| Requirement | Purpose |
| --- | --- |
| Node.js and npm | Install dependencies. |
| Docker | Build and push the image. |
| AWS CLI | Login to ECR. |
| Jenkins credential `aws-creds` | AWS access for ECR push. |
| Jenkins shared library `jenkins-shared-library` | Provides `nodeJSEKSPipeline`. |
| Pipeline Utility Steps plugin | Provides `readJSON`. |
| Workspace Cleanup plugin | Provides `cleanWs()`. |

## Current Notes

- `package.json` has an `npm test` command that exits with failure, but the current shared CI library does not run `npm test`; it runs `echo test`.
- CI pushes to ECR account `704887592073`.
- CI triggers the downstream job name dynamically as `../cart-CD`.