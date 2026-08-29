# Enterprise Todo Application (Monorepo)

A secure, scalable enterprise Todo application built with **Angular**, **Spring Boot**, **MongoDB Atlas**, **Redis**, **Elasticsearch**, **MinIO (S3)**, **Keycloak**, and **Nginx Reverse Proxy with HTTPS**.

---

## 📁 Repository Structure

```text
Todo-New/
├── docs/                    # Architecture, security guides, and migration notes
├── infrastructure/          # Docker Compose stack, Nginx config, TLS certs, and scripts
│   ├── keycloak/            # Pre-configured Keycloak realm (todo-realm.json)
│   ├── nginx/               # Nginx reverse proxy configuration & TLS cert scripts
│   ├── scripts/             # Local backup and utility scripts
│   ├── .env                 # Environment variables for the Docker stack
│   ├── docker-compose.yaml  # Orchestrates all 8 services + backup container
│   └── README.md            # Detailed infrastructure documentation
├── todo-backend/            # Spring Boot 4 REST API & business logic
│   ├── src/                 # Java source code (controllers, services, repositories)
│   ├── Dockerfile           # Multi-stage JDK 17 build & runtime container
│   └── pom.xml              # Maven dependencies
└── todo-frontend/           # Angular 21 Single Page Application
    ├── src/                 # TypeScript components, services, auth guards, styles
    ├── Dockerfile           # Node 22 runtime container
    └── package.json         # NPM dependencies
```

---

## 🏗️ Architecture & Services

All traffic enters securely through **Nginx (SSL Termination)** on ports `80` and `443`.

| Service | Container Name | Internal Port | Public URL | Description |
| :--- | :--- | :--- | :--- | :--- |
| **Nginx** | `todo-nginx` | `80`, `443` | `https://localhost` | HTTPS Reverse Proxy & SSL Termination |
| **Frontend** | `todo-frontend` | `4200` | `https://localhost/` | Angular 21 Single Page Application |
| **Backend** | `todo-backend` | `8081` | `https://localhost/api/...` | Spring Boot 4 Enterprise REST API |
| **Keycloak** | `todo-keycloak` | `8080` | `https://localhost/realms/...`<br>`https://localhost/admin/` | Identity & Access Management (OAuth2 / OIDC) |
| **MinIO** | `todo-minio` | `9000` (API)<br>`9001` (Console) | `https://minio.localhost/` | S3-compatible Object Storage with TLS & SSE-S3 encryption |
| **MongoDB** | `todo-mongodb` | `27017` | *Internal* | Primary database (or MongoDB Atlas cloud) |
| **Redis** | `todo-redis` | `6379` | *Internal* | In-memory cache for fast read operations |
| **Elasticsearch**| `todo-elasticsearch`| `9200` | *Internal* | Full-text search engine for tasks |

---

## 🚀 Quick Start (Running with Docker)

### Prerequisites

1. **Docker Desktop**: Must be installed and running on your machine.

---

### Step 1: Generate Local TLS Certificates

Before starting Docker, generate the self-signed SSL/TLS certificates for Nginx and MinIO:

```powershell
cd D:\Todo-New\infrastructure
.\nginx\generate-local-cert.ps1
```

> This creates:
> - `infrastructure/nginx/certs/localhost.crt` & `localhost.key` (Nginx HTTPS)
> - `infrastructure/nginx/certs/public.crt` & `private.key` (MinIO HTTPS)

---

### Step 2: Configure Environment Variables

Verify or create `infrastructure/.env`:

```env
MONGO_ROOT_USERNAME=admin
MONGO_ROOT_PASSWORD=adminpass
MONGO_DATABASE=tododb
REDIS_PASSWORD=redis123
SECURITY_ENABLED=true

MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
MINIO_KMS_SECRET_KEY=todo-local-key:SjO+jQawQblauUtBCzRrZbydQfXnnSVuJT1m3BLu6GY=

S3_ENCRYPTION_ENABLED=true
S3_ENCRYPTION_MODE=S3
S3_ENDPOINT=https://minio:9000
S3_REQUIRE_TLS=true

# Local MongoDB (default)
SPRING_DATA_MONGODB_URI=mongodb://admin:adminpass@mongodb:27017/tododb?authSource=admin
# Or MongoDB Atlas:
# SPRING_DATA_MONGODB_URI=mongodb+srv://<username>:<password>@cluster0.mongodb.net/tododb?retryWrites=true&w=majority
```

---

### Step 3: Build & Start All Services

From the `infrastructure` directory:

```powershell
cd D:\Todo-New\infrastructure
docker compose up -d --build
```

To stop all services:

```powershell
docker compose down
```

---

## 🌐 Application Access & Credentials

Once the containers are healthy, open your web browser:

| Application / Console | URL | Default Credentials |
| :--- | :--- | :--- |
| **Todo App (Frontend)** | [https://localhost](https://localhost) | User: `testuser` / Password: `password` |
| **Keycloak Admin Console** | [https://localhost/admin/](https://localhost/admin/) | User: `admin` / Password: `admin` |
| **MinIO Storage Console** | [https://minio.localhost](https://minio.localhost) | User: `minioadmin` / Password: `minioadmin` |

*(Note: In local development with self-signed certificates, click **Advanced -> Proceed to localhost** in your browser if prompted).*

---

## 🔍 Verification & Health Checks

### Check container status:
```powershell
docker compose ps
```

### View container logs:
```powershell
# All services
docker compose logs -f

# Specific service (e.g. backend or keycloak)
docker compose logs -f backend
docker compose logs -f keycloak
```

### Test HTTPS endpoint connectivity:
```powershell
# Frontend
curl.exe -k -I https://localhost

# Keycloak OpenID configuration
curl.exe -k https://localhost/realms/todo/.well-known/openid-configuration

# Backend API (returns 401 if security is enabled and no JWT provided)
curl.exe -k -I https://localhost/api/v1/todos
```

---

## 💾 Database Backups

To trigger an on-demand database backup:

```powershell
cd D:\Todo-New\infrastructure
docker compose --profile backup run --rm backup
```
Backups are saved to `infrastructure/backups/YYYY-MM-DD/dump/`.
