<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&height=190&color=0:0B1220,55:075985,100:0EA5E9&text=Three-Tier%20Containerized%20Platform&fontColor=FFFFFF&fontSize=34&fontAlignY=38&desc=.NET%20REST%20API%20%7C%20Next.js%20%7C%20Vue%203%20%7C%20MySQL%20%7C%20GitHub%20Actions&descAlignY=60&descSize=15" width="100%" alt="Three-Tier Containerized Platform" />

![DevOps](https://img.shields.io/badge/Focus-Cloud%20%26%20DevOps-0EA5E9?style=for-the-badge&logo=icloud&logoColor=white) ![IaC](https://img.shields.io/badge/Infrastructure-as%20Code-7B42BC?style=for-the-badge&logo=terraform&logoColor=white) ![Docs](https://img.shields.io/badge/Documentation-Architecture%20%7C%20Tradeoffs-334155?style=for-the-badge)

</div>

---

# Three-Tier Application — Docker and Automated EC2 Delivery

A full-stack monorepo with a .NET REST API, a Next.js customer-facing web app, a Vue 3 admin panel, and MySQL. Docker Compose provides a repeatable local stack; GitHub Actions builds/pushes three images and deploys them to an EC2 host over SSH.

![Docker](https://img.shields.io/badge/Docker-Containers-2496ED?logo=docker&logoColor=white) ![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-CI%2FCD-2088FF?logo=githubactions&logoColor=white) ![.NET](https://img.shields.io/badge/.NET-REST%20API-512BD4?logo=dotnet&logoColor=white) ![Next.js](https://img.shields.io/badge/Next.js-Web%20UI-black?logo=nextdotjs) ![Vue](https://img.shields.io/badge/Vue-Admin%20Panel-4FC08D?logo=vuedotjs&logoColor=white) ![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?logo=mysql&logoColor=white)

[**Source code**](https://github.com/AhmedRaza92110/docker-3tier-application-project) · [Deployment workflow](.github/workflows/deploy.yml) · [Architecture notes](docs/overview.md) · [Deployment guide](docs/deployment.md)

## 🏛️ Application architecture

```mermaid
flowchart TB
    Customer["Customer Browser"] --> Next["Next.js Frontend<br/>:3000"]
    Admin["Admin Browser"] --> Vue["Vue 3 Admin Panel<br/>:5173 external / :80 container"]
    Next -->|REST / JSON| API[".NET REST API<br/>:5005 in production compose"]
    Vue -->|REST / JSON| API
    API -->|EF Core / MySQL protocol| DB[("MySQL 8<br/>named volume")]
    classDef web fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
    classDef api fill:#ede9fe,stroke:#7c3aed,color:#4c1d95
    classDef data fill:#fef3c7,stroke:#d97706,color:#78350f
    class Next,Vue web
    class API api
    class DB data
```

## 🔁 CI/CD pipeline

```mermaid
flowchart TD
    Push["Push to main"] --> GHA["GitHub Actions runner"]
    GHA --> APIImage["Build and push .NET API image"]
    GHA --> WebImage["Build and push Next.js image"]
    GHA --> PanelImage["Build and push Vue panel image"]
    APIImage --> Registry[("Docker Hub")]
    WebImage --> Registry
    PanelImage --> Registry
    Registry --> Compose["Copy docker-compose.prod.yml to EC2"]
    Compose --> SSH["SSH deployment step"]
    SSH --> Pull["docker compose pull"]
    Pull --> Up["docker compose up -d --force-recreate"]
    Up --> Verify["Manual health/route verification"]
```

The workflow builds three images with the `latest` tag, then copies the production Compose file to EC2, pulls the images and recreates containers. It does **not currently include automated tests, image vulnerability scanning, immutable release tags, or an automated post-deploy smoke-test/rollback stage**.

## 🧩 Stack and infrastructure decisions

| Layer | Technology | Why |
|---|---|---|
| Customer UI | Next.js | Component-driven web experience |
| Admin UI | Vue 3 + Vite | Separate admin experience and independent build |
| API | ASP.NET Core REST API | Typed HTTP boundary for business operations |
| Database | MySQL 8 | Relational storage for products, users, orders and categories |
| Local orchestration | Docker Compose | One command starts the database and application tiers |
| Image distribution | Docker Hub | Separates image build from target-host runtime |
| Automation | GitHub Actions | Repeatable build, publish and deployment workflow |
| Runtime host | AWS EC2 | Simple, understandable target for a first automated deployment |

## ▶️ Run locally

Prerequisites: Docker Desktop with Compose.

```bash
docker compose up --build
```

Open:
- Next.js frontend: `http://localhost:3000`
- Vue admin panel: `http://localhost:5173`
- .NET Swagger/API: `http://localhost:5000/swagger`

Stop containers with `docker compose down`. Use `docker compose down -v` only if you intentionally want to delete the local MySQL volume and its data.

> **Important:** The local Compose file contains example development credentials and publishes the database port. Do not reuse those values in a public or production environment.

## 🔐 GitHub Actions secrets and EC2 setup

The workflow expects these repository Actions secrets:
- `DOCKER_PASSWORD` — Docker Hub access token.
- `EC2_HOST` — EC2 public DNS/IP.
- `EC2_USERNAME` — SSH user.
- `EC2_SSH_KEY` — private SSH key content.

The EC2 host needs Docker Engine and the Docker Compose plugin installed, outbound access to Docker Hub, and permission to run containers. Restrict SSH ingress to trusted IPs and never commit private keys or credentials.

## 💡 Architecture trade-offs

- **Single EC2 host:** inexpensive and easy to understand for a portfolio demo, but it is a single point of failure and does not provide multi-AZ high availability.
- **Compose on the host:** appropriate for a small deployment, but it is not a cluster scheduler.
- **Separate UI containers:** lets each UI be built/released independently, but requires a clear API URL strategy.
- **MySQL named volume:** retains data across container recreation on that host, but is not a backup or disaster-recovery strategy.
- **Image registry:** makes deployment pull the same packaged artifacts that CI produced; immutable SHA tags would improve traceability over `latest`.

## 🛡️ Production-hardening roadmap

- [ ] Replace default DB passwords and remove production fallback credentials.
- [ ] Use GitHub OIDC or scoped short-lived cloud credentials where practical; restrict EC2 SSH access.
- [ ] Build and deploy immutable commit-SHA tags, not only `latest`.
- [ ] Add .NET/frontend/panel tests, linting and Trivy image scans as required gates.
- [ ] Add health checks and post-deploy smoke tests; keep the previous known-good release for rollback.
- [ ] Put a TLS reverse proxy or ALB in front of the services; keep the database private.
- [ ] Move persistent production data to a managed database with backups and restore testing.
- [ ] Add logs, metrics, alerting and a documented teardown/recovery runbook.

## 📁 Repository layout

```text
apis/                    ASP.NET Core API and Dockerfile
frontend/                Next.js customer UI and Dockerfile
panel/                   Vue 3 admin UI and Dockerfile
database/init.sql        MySQL initialization
docker-compose.yml       Local development stack
docker-compose.prod.yml  EC2 image-based deployment stack
.github/workflows/       GitHub Actions build/push/deploy workflow
docs/                    Architecture and component guides
```

<div align="center"><sub>End-to-end application delivery practice with explicit trade-offs and next steps.</sub></div>
