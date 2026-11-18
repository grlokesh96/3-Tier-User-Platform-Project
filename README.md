# 3-Tier User Platform Project

A production-style **3-tier web application** for user management, built with a React frontend, an Express + MySQL backend, and a containerised deployment on Kubernetes (AWS EKS) with a full DevSecOps CI/CD pipeline.

The platform exposes a full CRUD API over a `users` table, serves a bundled SPA from the same Node process, and ships a hardened container image that is promoted automatically from QA to production.

---

## Table of Contents

- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Features](#features)
- [API Reference](#api-reference)
- [Database Schema](#database-schema)
- [Local Development](#local-development)
- [Docker](#docker)
- [Kubernetes Deployment](#kubernetes-deployment)
- [CI/CD Pipeline](#cicd-pipeline)
- [DevSecOps Scanning](#devsecops-scanning)
- [Environment Variables](#environment-variables)
- [Troubleshooting](#troubleshooting)

---

## Architecture

```
┌──────────────────────────────┐
│        Presentation Tier      │   React 17 SPA
│        (client/)              │   bundled by Webpack → client/public
└───────────────┬──────────────┘
                │  HTTP/JSON  (/api/users)
                ▼
┌──────────────────────────────┐
│        Application Tier       │   Express 4 (Node.js 20)
│        (server/)              │   routes → controllers → models
└───────────────┬──────────────┘
                │  mysql2 (parameterised SQL)
                ▼
┌──────────────────────────────┐
│          Data Tier            │   MySQL 8
│                              │   `users` table
└──────────────────────────────┘
```

The presentation and application tiers are packaged into a **single container image**. Webpack compiles the React client into `client/public`, and the Express server serves those static assets alongside the JSON API on port `5000`.

### Request flow

1. The browser loads `index.html` and `bundle.js` from the Express static handler.
2. The client calls `GET /api/users`; Express routes the request to the controller.
3. The controller delegates to the model, which runs a parameterised query via `mysql2`.
4. The JSON response is rendered as a user list; forms post back to the same API.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 17, React DOM, Axios, Webpack 5, Babel 7 |
| Backend | Node.js 20, Express 4, `mysql2`, CORS, body-parser |
| Database | MySQL 8 |
| Containerisation | Docker (`node:20-alpine` base) |
| Orchestration | Kubernetes (AWS EKS, `ap-south-1`) with AWS Load Balancer Controller |
| CI/CD | GitHub Actions |
| Security | Gitleaks, Trivy, Checkov, SonarQube, SPDX SBOM generation |
| Code quality | SonarQube Quality Gate |

---

## Project Structure

```
.
├── .dockerignore                     # Build context exclusions
├── Dockerfile                        # Builds client + server into one image
├── sonar-project.properties          # SonarQube project configuration
├── .github/
│   └── workflows/
│       ├── qa-cicd.yml               # QA CI/CD pipeline (build, scan, deploy)
│       └── prod-cd.yaml              # Production promotion pipeline
├── client/                           # ── Presentation tier ──
│   ├── package.json
│   ├── webpack.config.js             # Entry src/index.js → public/bundle.js
│   ├── .babelrc
│   ├── public/                       # index.html, bundle.js, styles
│   └── src/
│       ├── index.js                  # Vanilla-JS UI: load, create, edit, delete
│       ├── App.js                    # React component version of the UI
│       ├── App.css
│       ├── api/
│       │   └── users.js              # Axios API client
│       └── components/
│           ├── UsersList.js          # React list container
│           └── UserItem.js           # React row component
├── server/                           # ── Application tier ──
│   ├── package.json
│   ├── server.js                     # Entry point: DB init, routes, static serving
│   ├── app.js                        # Express app factory (router mount)
│   ├── config/
│   │   └── db.js                     # mysql2 connection from env vars
│   ├── routes/
│   │   ├── userRoutes.js             # Router → controller bindings
│   │   └── users.js                  # Standalone CRUD router
│   ├── controllers/
│   │   └── userController.js         # Request handling
│   └── models/
│       └── userModel.js              # Data access layer
└── k8-manifests/
    ├── qa/                           # QA namespace manifests
    │   ├── app-deployment.yaml
    │   ├── app-svc.yaml
    │   └── app-ingress.yaml
    └── prod/                         # Production namespace manifests
        ├── app-deployment.yaml
        ├── app-svc.yaml
        └── app-ingress.yaml
```

---

## Features

- **Full CRUD** over users — list, create, update, delete.
- **Two UI implementations** — a React component tree (`App.js`, `UsersList.js`, `UserItem.js`) and a dependency-free vanilla-JS entry point (`index.js`).
- **Self-initialising schema** — the `users` table is created on boot with `CREATE TABLE IF NOT EXISTS`, so no separate migration step is required.
- **Parameterised queries** throughout, preventing SQL injection.
- **CORS enabled** on the API for cross-origin client development.
- **SPA fallback routing** — unmatched non-API routes return `index.html`.
- **Non-root container** — the image runs as a dedicated `appuser` in `appgroup`.
- **Unified image** — one artifact is built once and promoted QA → prod, so production runs exactly what was tested.
- **Supply-chain visibility** — SBOMs (SPDX) are generated for both source and image on every build.

---

## API Reference

Base URL: `http://localhost:5000/api`

| Method | Endpoint | Description | Body / Params | Success |
|---|---|---|---|---|
| `GET` | `/api/users` | List all users | — | `200` — user array |
| `POST` | `/api/users` | Create a user | `{ name, email, role }` | `201` — created user |
| `PUT` | `/api/users/:id` | Update a user | `{ name, email, role }` | `200` — updated user |
| `DELETE` | `/api/users/:id` | Delete a user | — | `200` — confirmation message |

`role` must be one of `Admin` or `User` (the column is an `ENUM`). `email` must be unique.

### Example requests

```bash
# List users
curl http://localhost:5000/api/users

# Create a user
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Ada Lovelace","email":"ada@example.com","role":"Admin"}'

# Update a user
curl -X PUT http://localhost:5000/api/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Ada Lovelace","email":"ada@example.com","role":"User"}'

# Delete a user
curl -X DELETE http://localhost:5000/api/users/1
```

### Error responses

Database and validation failures return a JSON body of the form:

```json
{ "error": "<message>" }
```

with HTTP status `500`.

---

## Database Schema

The schema is created automatically at server start-up:

```sql
CREATE TABLE IF NOT EXISTS users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE,
  role ENUM('Admin', 'User') NOT NULL
);
```

In Kubernetes, credentials are supplied through a `mysql-secret` Secret in each namespace (`MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`, `DATABASE_URL` keys). See [Environment Variables](#environment-variables).

---

## Local Development

### Prerequisites

- Node.js 18+ (20 recommended)
- npm
- A running MySQL 8 instance

### 1. Start MySQL

```bash
docker run -d --name mysql \
  -e MYSQL_ROOT_PASSWORD=password123 \
  -e MYSQL_DATABASE=test_db \
  -p 3306:3306 \
  mysql:8
```

### 2. Run the server

```bash
cd server
npm install
npm start
```

The server listens on `http://localhost:5000` and creates the `users` table on boot.

### 3. Run the client

In a second terminal:

```bash
cd client
npm install
npm start     # webpack --mode development
```

Open the served `client/public/index.html`. The UI calls `/api/users` on the same origin, so the API is reached through the Express server's static path.

### 4. Build for production

```bash
cd client
npm run build   # webpack --mode production → client/public/bundle.js
```

---

## Docker

The `Dockerfile` builds both tiers into a single image:

1. `node:20-alpine` base with a `client` stage — installs client deps and runs the Webpack production build.
2. `server` stage — installs production-only server deps and copies the compiled client into `server/public`.
3. Creates a non-root `appuser`, sets `NODE_ENV=production`, and exposes port `5000`.

### Build and run

```bash
# Build
docker build -t devopsshack/nodejs-app:latest .

# Run
docker run -p 5000:5000 \
  -e DB_HOST=host.docker.internal \
  -e DB_USER=appuser \
  -e DB_PASSWORD=password123 \
  -e DB_NAME=test_db \
  devopsshack/nodejs-app:latest
```

The image is published to Docker Hub as `devopsshack/nodejs-app` with two tags: the commit SHA and `latest`.

---

## Kubernetes Deployment

Manifests live under `k8-manifests/`, split by environment. Each environment has a Deployment, a ClusterIP Service, and an AWS ALB Ingress.

| Environment | Namespace | Host | Image tag |
|---|---|---|---|
| QA | `qa` | `my-app.art` | `devopsshack/nodejs-app:<sha>` |
| Production | `prod` | `adityajaiswal.com` | `devopsshack/nodejs-app:prod-<sha>` |

### Deploy manually

```bash
aws eks update-kubeconfig --region ap-south-1 --name my-cluster

kubectl apply -f k8-manifests/qa/ -n qa
kubectl rollout status deployment/nodejs-app -n qa --timeout=180s
```

### Cluster prerequisites

- An **AWS Load Balancer Controller** installed, with the `alb` ingress class.
- A `regcred` image pull secret in the target namespace (for Docker Hub).
- A `mysql-secret` Secret in the target namespace with the keys listed above.
- A MySQL service reachable at the hostname `mysql` within the cluster.

### Verify

```bash
kubectl get pods -n qa
kubectl get svc -n qa
kubectl get ingress -n qa
```

---

## CI/CD Pipeline

Two GitHub Actions workflows run on pushes to `main`.

### QA pipeline — `.github/workflows/qa-cicd.yml`

Runs when `client/**`, `server/**`, or the workflow file itself changes:

```
gitleaks-scan
   │
   ├── checkov-terraform ─┐
   ├── checkov-k8s ───────┤
   ├── checkov-dockerfile ┤
   ├── client-trivy-fs ───┤
   └── server-trivy-fs ───┘
          │
     client-lint / server-lint
          │
     client-test / server-test
          │
       sonarqube  (scan + Quality Gate)
          │
     client-build
          │
        docker  (build → image scan → SBOM → push)
          │
   update-qa-manifest  (rewrite image tag, commit to `qa`)
          │
     deploy-to-k8s  (EKS `qa` namespace via OIDC)
```

### Production pipeline — `.github/workflows/prod-cd.yaml`

Runs when `k8-manifests/prod/app-deployment.yaml` changes:

```
promote-image-to-prod   (pull :latest, retag prod-<sha>, push)
          │
   update-prod-manifest  (rewrite prod image tag, commit to `main`)
          │
     deploy-to-k8s  (EKS `prod` namespace via OIDC)
```

### Required secrets and variables

| Name | Type | Purpose |
|---|---|---|
| `SONAR_TOKEN` | Secret | SonarQube authentication |
| `SONAR_HOST_URL` | Secret | SonarQube server URL |
| `DOCKERHUB_TOKEN` | Secret | Docker Hub push credentials |
| `DOCKERHUB_USERNAME` | Variable | Docker Hub username |
| `AWS_ROLE_TO_ASSUME` | Secret | OIDC role for the QA EKS cluster |
| `AWS_ROLE_TO_ASSUME1` | Secret | OIDC role for the production EKS cluster |

Both workflows request `id-token: write` so AWS credentials are obtained via **OIDC** — no static access keys are stored in the repository.

---

## DevSecOps Scanning

Every pipeline run produces the following artifacts:

| Tool | Scope | Output |
|---|---|---|
| **Gitleaks** | Full repository | `gitleaks-report.json` — secret detection, values redacted |
| **Trivy (FS)** | `client/`, `server/` | JSON + table reports for `HIGH`/`CRITICAL` vulnerabilities |
| **Trivy (Image)** | Built Docker image | JSON + table reports for `HIGH`/`CRITICAL` vulnerabilities |
| **Checkov** | Terraform, Kubernetes, Dockerfile | JSON reports for IaC misconfigurations |
| **SonarQube** | Full repository | Code quality metrics + Quality Gate status |
| **Anchore SBOM** | Source and image | SPDX JSON Software Bill of Materials |

Scans run with `--exit-code 0` so findings are reported as artifacts rather than hard-failing the build; review them in the Actions tab. The SonarQube Quality Gate result is echoed into the job summary.

---

## Environment Variables

| Variable | Default | Used by | Description |
|---|---|---|---|
| `PORT` | `5000` | server | HTTP listen port |
| `DB_HOST` | `localhost` (`mysql`) | server | MySQL hostname |
| `DB_USER` | `appuser` (`root` in `config/db.js`) | server | MySQL username |
| `DB_PASSWORD` | `password123` (`password` in `config/db.js`) | server | MySQL password |
| `DB_NAME` | `test_db` | server | Database name |
| `NODE_ENV` | — | Docker | Set to `production` in the image |

> **Note:** `server/server.js` and `server/config/db.js` ship different fallback defaults. Set the variables explicitly rather than relying on the fallbacks.

---

## Troubleshooting

**`Database connection failed`**
MySQL is unreachable. Verify the server is running, that `DB_HOST` resolves, and that the credentials match. The server exits with code `1` on a failed connection by design.

**`ER_DUP_ENTRY` on create**
The `email` column is `UNIQUE`. Each user needs a distinct email address.

**`ER_DATA_TOO_LONG` on create**
`name` and `email` are capped at 255 characters.

**Client renders but the API call fails**
The client calls `/api/users` on the same origin. Ensure the Express server is running and reachable, or update `API_URL` in `client/src/api/users.js` to point at the server.

**Port 5000 already in use**
Set `PORT` to a free port, and update the `containerPort` in the Deployment and `targetPort` in the Service to match.

**Kubernetes pod is `ImagePullBackOff`**
The `regcred` pull secret is missing from the namespace, or the image tag in the manifest does not exist in Docker Hub.

**Deployment does not become ready**
Check `kubectl describe pod -n <namespace> <pod>` and `kubectl logs -n <namespace> <pod>` — a database connection failure at boot will crash-loop the container.
