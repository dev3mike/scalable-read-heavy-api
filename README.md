# Scalable Read-Heavy API

A production-ready NestJS API built to handle **read-heavy workloads** at scale. Designed for applications where reads far outnumber writes — like product catalogs, news feeds, dashboards, and public APIs.

---

## What Makes This "Read-Heavy" Scalable?

Most web applications read data way more than they write it. For example, an e-commerce site might get 1,000 people browsing products for every 1 person placing an order. This project is built around that pattern:

```
                         ┌──────────────────────┐
              writes     │   1 Primary DB       │
             ┌──────────>│   (INSERT/UPDATE)     │
             │           └──────────┬────────────┘
             │                      │ replication
  ┌──────────┴──────────┐           │
  │   5-20 API Pods     │  ┌────────▼────────────┐
  │   (auto-scaled)     │  │  10 Read Replicas   │
  │                     │  │  (SELECT queries)   │
  └──────────┬──────────┘  │  load balanced      │
             │             └─────────────────────┘
             │  reads            ▲
             └───────────────────┘
```

- **Writes** go to a single primary database
- **Reads** are spread across **10 replicas** — each replica shares the load
- **API pods** auto-scale from 5 to 20 based on traffic
- A **hot standby** database is always ready to take over if the primary goes down

---

## How Many Users Can This Handle?

In production (with auto-scaling enabled):

| Usage Pattern | Concurrent Users |
|---|---|
| Light (browsing, 1 request every 10s) | **50,000 - 100,000** |
| Medium (active use, 1 request every 3s) | **15,000 - 30,000** |
| Heavy (real-time, 1 request per second) | **5,000 - 10,000** |

> Adding a Redis cache layer can multiply these numbers by 5-10x.

---

## Tech Stack

| Component | Technology |
|---|---|
| API Framework | [NestJS](https://nestjs.com/) (Node.js + TypeScript) |
| Database | PostgreSQL 16 |
| DB Operator (prod) | [CloudNativePG](https://cloudnative-pg.io/) |
| Container Orchestration | Kubernetes |
| Config Management | Kustomize (built into kubectl) |
| Secrets | AWS Secrets Manager + [External Secrets Operator](https://external-secrets.io/) |

---

## Project Structure

```
scalable-read-heavy-api/
├── src/                     # Application source code (NestJS)
│   ├── (...)
├── test/                    # Tests
├── k8s/                     # Kubernetes configs
│   ├── base/                #   Shared configs (all environments)
│   │   ├── deployment.yaml  #     API deployment with health checks
│   │   ├── service.yaml     #     ClusterIP service
│   │   ├── hpa.yaml         #     Auto-scaling rules
│   │   ├── postgres/        #     Single-instance PostgreSQL
│   │   └── ...
│   ├── components/         #   Optional components (external-secrets for test/stage/prod)
│   └── overlays/            #   Per-environment overrides
│       ├── local/          #     1 pod, 1 DB, no AWS — run on Kind out-of-the-box
│       ├── test/            #     2 pods, 1 DB, minimal resources
│       ├── stage/           #     3 pods, 1 DB, moderate resources
│       └── prod/            #     5-20 pods, 12-node DB cluster
├── Dockerfile               # Container image for API
├── kind-config.yaml         # Kind cluster configuration for local testing
├── package.json
├── tsconfig.json
└── nest-cli.json
```

---

## Getting Started

### Prerequisites

- Node.js 18+
- npm

### Install and Run Locally

```bash
# Install dependencies
npm install

# Start in development mode (with hot reload)
npm run start:dev

# Run tests
npm run test

# Build for production
npm run build

# Start production server
npm run start:prod
```

The API starts on `http://localhost:3000` by default.

---

## Environments

The app runs in multiple environments, each with its own Kubernetes namespace, database, and secrets:

| | Local | Test | Stage | Prod |
|---|---|---|---|---|
| **Purpose** | Run on Kind with no AWS required | Lightweight shared env | Pre-prod validation | Production |
| **API Pods** | 1 | 2 | 3 | 5 (auto-scales to 20) |
| **Database** | 1 instance | 1 instance | 1 instance | 12-node cluster |
| **Secrets** | K8s Secret (generated) | AWS Secrets Manager | AWS Secrets Manager | AWS Secrets Manager |

---

## Testing Locally with Kind

You can test the entire Kubernetes setup locally using [Kind](https://kind.sigs.k8s.io/) (Kubernetes in Docker).

### Prerequisites

1. [Docker](https://www.docker.com/) installed and running
2. [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) installed
3. `kubectl` installed

### Quick Start

```bash
# Option 1: All-in-one command (creates cluster, builds image, loads it, and deploys)
npm run k8s:up:local

# Option 2: Step by step
# 1. Create a local Kubernetes cluster (1 control-plane + 2 worker nodes)
npm run k8s:kind:create

# 2. Build Docker image and load it into Kind
npm run k8s:build-and-load

# 3. Deploy to local environment (no AWS / External Secrets required)
npm run k8s:deploy:local

# 4. Check status
npm run k8s:status:local

# 5. Forward port to access the API (in a separate terminal)
npm run k8s:port-forward:local

# 6. View logs
npm run k8s:logs:local

# When done, delete the cluster
npm run k8s:kind:delete
```

### Local Testing Workflow

```bash
# 1. Create cluster (if not already created)
npm run k8s:kind:create

# 2. Build Docker image and load it into Kind
npm run k8s:build-and-load

# 3. Deploy application (local overlay generates secrets automatically)
npm run k8s:deploy:local

# 4. Wait for pods to be ready, then access the API
npm run k8s:pods:local       # Check pod status
npm run k8s:port-forward:local  # Forward port 3000 (run in separate terminal)

# 5. Monitor deployment
npm run k8s:db:local         # Check database
npm run k8s:logs:local       # Stream API logs

# 6. Clean up
npm run k8s:delete:local     # Remove application
npm run k8s:kind:delete      # Remove cluster
```

> **Note:** The `local` overlay does not use External Secrets. It generates an `api-secrets` Kubernetes Secret with default values so it works out-of-the-box.

---

## Deploying to Kubernetes

### Prerequisites

- **All environments:** A Kubernetes cluster (e.g. [Kind](https://kind.sigs.k8s.io/) for local) and `kubectl` configured.
- **Local only:** Nothing else — the local overlay uses in-cluster secrets and the base Postgres StatefulSet.
- **Test / stage / prod:** [External Secrets Operator](https://external-secrets.io/) installed; [CloudNativePG Operator](https://cloudnative-pg.io/) for **prod** only.

### Store Secrets in AWS (test / stage / prod only; not needed for local)

```bash
# Create database credentials for each environment
aws secretsmanager create-secret \
  --name "scalable-read-heavy-api/prod/db" \
  --secret-string '{"username":"app","password":"your-secure-password"}'

aws secretsmanager create-secret \
  --name "scalable-read-heavy-api/prod/app" \
  --secret-string '{"jwt_secret":"your-jwt-secret"}'
```

### Deploy

Using npm scripts (recommended):

```bash
# Deploy to local (Kind; no AWS required — build/load image first: npm run k8s:build-and-load)
npm run k8s:deploy:local

# Deploy to test
npm run k8s:deploy:test

# Deploy to staging
npm run k8s:deploy:stage

# Deploy to production
npm run k8s:deploy:prod
```

Or using kubectl directly:

```bash
# Deploy to local
kubectl apply -k k8s/overlays/local

# Deploy to test
kubectl apply -k k8s/overlays/test

# Deploy to staging
kubectl apply -k k8s/overlays/stage

# Deploy to production
kubectl apply -k k8s/overlays/prod
```

### Verify

Using npm scripts (use `local`, `test`, `stage`, or `prod` for `{env}`):

```bash
# Check everything is running
npm run k8s:status:{env}

# Check pods
npm run k8s:pods:{env}

# Check auto-scaler status
npm run k8s:hpa:{env}

# Check database (local/test/stage: StatefulSet; prod: cluster)
npm run k8s:db:{env}

# View API logs (follow mode)
npm run k8s:logs:{env}

# Describe deployment
npm run k8s:describe:{env}
```

Or using kubectl directly:

```bash
# Check everything is running (use namespace: scalable-read-heavy-api-local, -test, -stage, or -prod)
kubectl get all -n scalable-read-heavy-api-local

# Check auto-scaler status
kubectl get hpa -n scalable-read-heavy-api-local

# Check database (local/test/stage: StatefulSet; prod: cluster)
kubectl get statefulset,svc -n scalable-read-heavy-api-local | grep postgres
# prod: kubectl get cluster -n scalable-read-heavy-api-prod

# View API logs
kubectl logs -n scalable-read-heavy-api-local -l component=api --tail=50
```

> For detailed Kubernetes documentation, see the [k8s/README.md](k8s/README.md).

---

## Secrets Management

Sensitive data never lives in config files. Here's how it works:

| Type | Stored In | Examples |
|---|---|---|
| **Secrets (local)** | Kubernetes Secret (generated by Kustomize) | DB credentials, JWT — no AWS needed |
| **Secrets (test/stage/prod)** | AWS Secrets Manager | DB credentials, JWT secret |
| **Config** | Kubernetes ConfigMaps | NODE_ENV, PORT, DB_HOST, LOG_LEVEL |

The **local** overlay generates an `api-secrets` Secret with default values so it runs without AWS. For **test**, **stage**, and **prod**, the External Secrets Operator syncs from AWS; each environment uses an isolated path (`scalable-read-heavy-api/test/db`, `scalable-read-heavy-api/stage/db`, `scalable-read-heavy-api/prod/db`, etc).

---

## Setting AWS Credentials

### On your machine (AWS CLI, scripts, storing secrets)

Use this when running `aws` commands (e.g. to create secrets in Secrets Manager) or any SDK on your laptop/CI.

**Option 1: AWS CLI (recommended)**

```bash
aws configure
# Enter: AWS Access Key ID, Secret Access Key, default region (e.g. us-east-1)
```

**Option 2: Environment variables**

```bash
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_REGION=us-east-1
# If using temporary credentials (e.g. SSO):
# export AWS_SESSION_TOKEN=...
```

**Option 3: Config file**

Edit `~/.aws/credentials`:

```ini
[default]
aws_access_key_id = AKIA...
aws_secret_access_key = ...
```

And `~/.aws/config` for region:

```ini
[default]
region = us-east-1
```

---

### In the cluster (so External Secrets can fetch from AWS)

The cluster needs permission to call AWS Secrets Manager. Two approaches:

#### EKS (production / staging)

Use **IRSA** (IAM Roles for Service Accounts). The base config expects the `api-sa` service account to have an IAM role attached.

1. Create an IAM role with a trust policy that allows your EKS OIDC provider to assume it.
2. Attach a policy that allows `secretsmanager:GetSecretValue` (and `kms:Decrypt` if secrets are encrypted with a KMS key) on the secrets you use.
3. Associate the role with the `api-sa` service account in each namespace (e.g. `scalable-read-heavy-api-prod`):

   ```bash
   kubectl annotate serviceaccount api-sa -n scalable-read-heavy-api-prod \
     eks.amazonaws.com/role-arn=arn:aws:iam::ACCOUNT_ID:role/YOUR_EXTERNAL_SECRETS_ROLE
   ```

No access keys are stored in the cluster; the pod gets temporary credentials via the service account.

#### Kind / local (your own credentials in the cluster)

For local clusters (e.g. Kind) where IRSA is not available, you can use **static credentials** stored in a Kubernetes Secret.

1. **Create a Secret** in the test namespace with your AWS credentials:

   ```bash
   kubectl create secret generic aws-credentials -n scalable-read-heavy-api-test \
     --from-literal=access-key-id=YOUR_AWS_ACCESS_KEY_ID \
     --from-literal=secret-access-key=YOUR_AWS_SECRET_ACCESS_KEY
   ```

2. **Use the SecretStore patch** so the test overlay uses these credentials instead of IRSA. Add this to `k8s/overlays/test/kustomization.yaml` under `patches:`:

   ```yaml
   patches:
     - path: patches/deployment-patch.yaml
     - path: patches/postgres-patch.yaml
     - path: patches/hpa-patch.yaml
     - path: patches/external-secret-patch.yaml
     - path: patches/secret-store-patch.yaml   # add this line
   ```

3. **Deploy** the test overlay as usual:

   ```bash
   npm run k8s:deploy:test
   ```

The patch is in `k8s/overlays/test/patches/secret-store-patch.yaml`. It points the SecretStore at the `aws-credentials` Secret. Use this only for local/Kind; in EKS use IRSA and do not add this patch.

---

## Production Safety

| Feature | Purpose |
|---|---|
| **Pod Disruption Budget** | At least 3 API pods always running during maintenance |
| **Network Policy** | Only API pods can reach the database |
| **Pod Anti-Affinity** | Pods spread across different nodes |
| **Auto-Failover** | Standby DB promotes in < 30 seconds if primary dies |
| **Auto Backups** | Daily database backups to S3, retained for 30 days |
| **Rolling Updates** | Zero-downtime deployments |
| **Health Probes** | Startup, readiness, and liveness checks on every pod |
| **HPA Stabilization** | 5-minute cooldown before scaling down (prevents flapping) |

---

## Available Scripts

### Development

| Command | Description |
|---|---|
| `npm run start` | Start the app |
| `npm run start:dev` | Start with hot reload |
| `npm run start:prod` | Start production build |
| `npm run build` | Build the project |
| `npm run test` | Run unit tests |
| `npm run test:watch` | Run tests in watch mode |
| `npm run test:e2e` | Run end-to-end tests |
| `npm run test:cov` | Run tests with coverage |
| `npm run lint` | Lint and fix code |
| `npm run format` | Format code with Prettier |

### Docker

| Command | Description |
|---|---|
| `npm run docker:build` | Build Docker image |
| `npm run docker:load:kind` | Load Docker image into Kind cluster |
| `npm run k8s:build-and-load` | Build image and load into Kind (combines both) |

### Kubernetes - Kind Cluster

| Command | Description |
|---|---|
| `npm run k8s:kind:create` | Create local Kubernetes cluster with Kind |
| `npm run k8s:kind:delete` | Delete local Kubernetes cluster |
| `npm run k8s:up:local` | Create Kind cluster (if needed) + build/load image + deploy |

### Kubernetes - Deployment

| Command | Description |
|---|---|
| `npm run k8s:deploy:local` | Deploy to local environment (no AWS required) |
| `npm run k8s:deploy:test` | Deploy to test environment |
| `npm run k8s:deploy:stage` | Deploy to staging environment |
| `npm run k8s:deploy:prod` | Deploy to production environment |
| `npm run k8s:delete:local` | Delete local environment |
| `npm run k8s:delete:test` | Delete test environment |
| `npm run k8s:delete:stage` | Delete staging environment |
| `npm run k8s:delete:prod` | Delete production environment |

### Kubernetes - Monitoring

| Command | Description |
|---|---|
| `npm run k8s:status:{env}` | Get all resources in namespace (local/test/stage/prod) |
| `npm run k8s:pods:{env}` | Get pods status (local/test/stage/prod) |
| `npm run k8s:logs:{env}` | Stream API logs in follow mode (local/test/stage/prod) |
| `npm run k8s:hpa:{env}` | Check Horizontal Pod Autoscaler status (local/test/stage/prod) |
| `npm run k8s:db:{env}` | Check database status (local/test/stage/prod) |
| `npm run k8s:describe:{env}` | Describe deployments (local/test/stage/prod) |

**Examples:**
```bash
npm run k8s:status:local   # Check local (Kind) status
npm run k8s:status:prod    # Check production status
npm run k8s:logs:test      # Stream test environment logs
npm run k8s:hpa:stage      # Check staging autoscaler
```

### Kubernetes - Port Forwarding

| Command | Description |
|---|---|
| `npm run k8s:port-forward:local` | Forward local port 3000 to local API service |
| `npm run k8s:port-forward:test` | Forward local port 3000 to test API service |
| `npm run k8s:port-forward:stage` | Forward local port 3000 to staging API service |
| `npm run k8s:port-forward:prod` | Forward local port 3000 to production API service |

**Example:**
```bash
# Deploy local environment
npm run k8s:deploy:local

# Forward port to access the API
npm run k8s:port-forward:local

# Then open http://localhost:3000 in your browser (e.g. http://localhost:3000/health)
```

---

## License

This project is [MIT licensed](https://github.com/nestjs/nest/blob/master/LICENSE).
