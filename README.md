# Kubernetes EKS Microservices Deployment (Docker, AWS EKS, RDS MySQL)

A full-stack 3-tier application deployed as containerized microservices on AWS EKS, with images shipped through Docker Hub and persistent data on RDS MySQL.

---

## Architecture

```
            Internet
               │
               ▼
        [AWS LoadBalancer]
               │
               ▼
     Frontend Pod (Next.js)        <-- EKS
               │
     (HTTP to public LoadBalancer)
               │
               ▼
        [AWS LoadBalancer]
               │
               ▼
     Backend Pod (Node.js)         <-- EKS
               │
               ▼
          AWS RDS MySQL
```

| Tier     | Technology         | Hosting          |
| -------- | ------------------ | ---------------- |
| Frontend | Next.js (React)    | EKS (Kubernetes) |
| Backend  | Node.js / Express  | EKS (Kubernetes) |
| Database | MySQL              | AWS RDS          |

## Features

- Browse books
- User registration and login (JWT auth)
- Submit and view book reviews

## Tech Stack

- **Frontend:** Next.js, React, Tailwind CSS
- **Backend:** Node.js, Express, Sequelize ORM
- **Database:** MySQL (AWS RDS)
- **Containerization:** Docker (multi-stage builds, linux/amd64)
- **Orchestration:** Kubernetes (AWS EKS via eksctl)
- **Image Registry:** Docker Hub

## Project Structure

```
.
├── backend/
│   ├── Dockerfile               # Multi-stage production build
│   ├── .dockerignore
│   └── src/
│       ├── server.js
│       ├── config/db.js
│       ├── models/
│       ├── controllers/
│       └── routes/
│
├── frontend/
│   ├── Dockerfile               # Multi-stage production build
│   ├── .dockerignore
│   └── src/app/
│       ├── page.js              # Books listing
│       ├── login/
│       ├── register/
│       └── book/[id]/           # Book detail + reviews
│
├── k8s/
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   ├── configmap.yaml
│   └── secret.yaml
│
└── docker-compose.yml           # Local development
```

## Container Registry Flow (Docker Hub)

EKS nodes pull images from Docker Hub at pod start. The flow is push-from-laptop, pull-from-cluster:

```
   Your laptop                Docker Hub                  EKS Worker Node
  ┌────────────┐           ┌──────────────┐            ┌──────────────────┐
  │ docker     │  push     │ drdre4664/   │   pull     │ kubelet pulls    │
  │ build      ├──────────▶│ bookreview-* │◀───────────┤ image on Pod     │
  │ + tag      │           │ :v1, :v2 ... │            │ create / restart │
  └────────────┘           └──────────────┘            └──────────────────┘
```

### Why a registry is required
Kubernetes does not build images — it only runs them. The `image:` field in each Deployment must point to a registry the cluster can reach. Docker Hub gives EKS nodes a public pull endpoint without extra networking setup.

### Auth
- Public repos: no pull secret needed; EKS pulls anonymously.
- Private repos: create a `docker-registry` Secret and reference it via `imagePullSecrets` in the Deployment.

### Immutable tag strategy
Each build is tagged with an explicit version (e.g. `:v3`) rather than `:latest`. This guarantees that `kubectl rollout restart` and `kubectl set image` produce deterministic deploys and that rollbacks point to a known artifact.

### How the Deployment references the image
```yaml
containers:
  - name: backend
    image: drdre4664/bookreview-backend:v3   # pulled from Docker Hub
    ports:
      - containerPort: 5000
```

## Deployment Steps

The order matters — the frontend needs the backend's public LoadBalancer URL before it is built, so the backend is shipped first.

1. **Provision the EKS cluster** with `eksctl` (VPC, node group, IAM).
2. **Create the RDS MySQL instance** and capture the endpoint, user, and password.
3. **Build and push the backend image** to Docker Hub (`docker build --platform linux/amd64 -t drdre4664/bookreview-backend:v3 . && docker push ...`).
4. **Apply backend ConfigMap/Secret** (DB host, user, password) and the backend Deployment + Service. Wait for the LoadBalancer to get an external hostname.
5. **Bake the backend URL into the frontend build** (`NEXT_PUBLIC_API_URL=http://<backend-lb>`) and push the frontend image to Docker Hub.
6. **Apply the frontend Deployment + Service** and wait for its LoadBalancer.
7. **Fix CORS** on the backend: set `CORS_ORIGIN` to the frontend LoadBalancer URL, `kubectl rollout restart deployment/backend`.
8. **Verify end-to-end**: open the frontend LB URL, register, log in, post a review, confirm the row in RDS.

## Local Development

```
docker-compose up --build
```

Brings up frontend, backend, and a local MySQL container with the same env contract as production.
