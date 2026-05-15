# github-contribution-finder

> AI-powered tool to discover GitHub repos to contribute to, deployed on k3s with a full CI/CD pipeline via GitHub Actions.

![CI/CD](https://github.com/Cookkiieess/github-contribution-finder/actions/workflows/ci-cd.yml/badge.svg)
![k3s](https://img.shields.io/badge/deployed%20on-k3s-blue)
![Docker](https://img.shields.io/badge/containerized-Docker-2496ED)
![Gemini](https://img.shields.io/badge/AI-Gemini_2.0_Flash-4285F4?logo=googlegemini&logoColor=white)

---

## What it does

Type what you want to work on in plain English — *"I want to add CI/CD pipelines to repos that don't have one"* or *"find beginner-friendly DevSecOps projects"* — and Gemini parses your intent, searches GitHub, and returns ranked repos that genuinely match your contribution goal with an explanation for each.

The Gemini API key is never exposed to the browser. All AI calls go through a lightweight Express backend running in the same k3s cluster, which holds the key securely as a Kubernetes secret.

---

## Architecture

```
Browser (React)
      │
      │  POST /api/search
      ▼
Express Server (Node.js)    ← GEMINI_API_KEY lives here as a k8s secret
      │
      ├── calls Gemini 2.0 Flash  (intent parsing + repo scoring)
      └── calls GitHub REST API   (repo search)
            │
            ▼
      Response → React UI

─────────────────────────────────────────────

Developer pushes code
        │
        ▼
  GitHub Repository
        │
        ▼
GitHub Actions (self-hosted runner on homelab)
  ├── Lint (ESLint)
  ├── Build client (Vite)
  ├── Docker build — client image (Nginx)
  ├── Docker build — server image (Node)
  ├── Push both images → local registry
  └── kubectl rollout → k3s cluster
        │
        ▼
  k3s Cluster (homelab)
  ├── client Deployment  →  Service  →  Ingress (Nginx)
  └── server Deployment  →  Service (ClusterIP, internal only)
        │
        ▼
  https://finder.homelab.local
```

The GitHub Actions workflow runs on a **self-hosted runner** inside the homelab so it can reach the private k3s cluster and local Docker registry directly — no port forwarding or VPN tunneling needed.

---

## CI/CD Pipeline

Each push to `main` triggers the following stages in sequence:

| Stage | What it does |
|---|---|
| **Lint** | ESLint across `client/src/` — fails fast on errors |
| **Build** | Vite production build of the React app |
| **Docker build (client)** | Multi-stage — Node compiles, Nginx serves the `dist/` output |
| **Docker build (server)** | Node alpine image for the Express API |
| **Push** | Both images pushed to the local homelab Docker registry |
| **Deploy** | `kubectl set image` rolls out both deployments in k3s |

Pull requests run lint + build only. Full deploy only triggers on merge to `main`.

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite |
| Backend | Node.js, Express |
| AI | Google Gemini 2.0 Flash API |
| Data | GitHub REST API |
| Containers | Docker (multi-stage builds) |
| Orchestration | k3s (Kubernetes) |
| CI/CD | GitHub Actions + self-hosted runner |
| Ingress | Nginx ingress controller |
| Secrets | Kubernetes secrets |
| OS | Ubuntu 24.04 |

---

## Project structure

```
github-contribution-finder/
├── .github/
│   └── workflows/
│       └── ci-cd.yml            # Full lint → build → push → deploy pipeline
├── k8s/
│   ├── deployment.yaml          # client + server deployments, resource limits
│   ├── service.yaml             # client NodePort, server ClusterIP
│   └── ingress.yaml             # Nginx ingress — routes / to client, /api to server
├── client/                      # React frontend
│   ├── src/
│   │   ├── App.jsx              # Main component
│   │   └── main.jsx             # Entry point
│   ├── index.html
│   └── package.json
├── server/                      # Express backend
│   ├── index.js                 # API routes — /api/search
│   └── package.json
├── Dockerfile.client            # Multi-stage: node:alpine → nginx:alpine
├── Dockerfile.server            # node:alpine, runs Express
├── nginx.conf                   # Nginx config for serving React SPA
├── .eslintrc.json               # ESLint rules
└── README.md
```

---

## Running locally

**Prerequisites:** Node 18+, a Gemini API key (free at [aistudio.google.com](https://aistudio.google.com))

```bash
git clone https://github.com/Cookkiieess/github-contribution-finder
cd github-contribution-finder

# Start the backend
cd server
npm install
echo "GEMINI_API_KEY=AIza..." > .env
node index.js
# → http://localhost:3001

# In a new terminal, start the frontend
cd client
npm install
npm run dev
# → http://localhost:5173
```

**Running with Docker:**

```bash
# Build and run the server
docker build -f Dockerfile.server -t gcf-server .
docker run -p 3001:3001 -e GEMINI_API_KEY=AIza... gcf-server

# Build and run the client
docker build -f Dockerfile.client -t gcf-client .
docker run -p 8080:80 gcf-client
# → http://localhost:8080
```

---

## Deploying to k3s

**1. Register the self-hosted GitHub Actions runner** on your homelab machine:

```bash
# GitHub repo → Settings → Actions → Runners → New self-hosted runner
# Follow the instructions shown — downloads and registers the runner agent
```

**2. Store your Gemini API key as a Kubernetes secret:**

```bash
kubectl create secret generic app-secrets \
  --from-literal=GEMINI_API_KEY=AIza...
```

**3. Apply the Kubernetes manifests:**

```bash
kubectl apply -f k8s/
```

**4. Push to `main`** — the pipeline handles everything from there.

---

## Self-hosted runner setup

The runner needs `docker`, `kubectl`, and `kubeconfig` access on the homelab machine:

```bash
# Verify the runner can reach your cluster
kubectl get nodes

# Verify Docker access
docker info
```

The Actions workflow uses the `self-hosted` label to route jobs to your runner.

---

## Environment variables

| Variable | Where | Description |
|---|---|---|
| `GEMINI_API_KEY` | k8s secret → server pod env | Gemini API key — never reaches the browser |
| `REGISTRY` | GitHub Actions secret | Local registry URL e.g. `registry.homelab:5000` |
| `KUBE_CONFIG` | GitHub Actions secret | Base64-encoded kubeconfig for kubectl in CI |

---

## License

MIT