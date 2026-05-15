# github-contribution-finder

> AI-powered tool to discover GitHub repos to contribute to, deployed on k3s with a full CI/CD pipeline via GitHub Actions.

![CI/CD](https://github.com/YOUR_USERNAME/github-contribution-finder/actions/workflows/ci-cd.yml/badge.svg)
![k3s](https://img.shields.io/badge/deployed%20on-k3s-blue)
![Docker](https://img.shields.io/badge/containerized-Docker-2496ED)

---

## What it does

Type what you want to work on in plain English — *"I want to add CI/CD pipelines to repos that don't have one"* or *"find beginner-friendly DevSecOps projects"* — and Claude parses your intent, searches GitHub, and returns ranked repos that genuinely match your contribution goal with an explanation for each.

---

## Architecture

```
Developer pushes code
        │
        ▼
  GitHub Repository
        │
        ▼
GitHub Actions (CI/CD)
  ├── Lint (ESLint)
  ├── Build (Vite)
  ├── Docker image build
  ├── Push → local registry (registry.homelab:5000)
  └── kubectl rollout → k3s cluster
        │
        ▼
  k3s Cluster (homelab)
  ├── Deployment (2 replicas)
  ├── Service (ClusterIP)
  └── Ingress (Nginx)
        │
        ▼
  https://finder.homelab.local
```

The GitHub Actions workflow runs on a **self-hosted runner** inside the homelab so it can reach the private k3s cluster and local Docker registry directly — no port forwarding or VPN tunneling needed.

---

## CI/CD Pipeline

Each push to `main` triggers the following stages:

| Stage | What it does |
|---|---|
| **Lint** | Runs ESLint across `src/` — fails fast on errors |
| **Build** | Vite production build, outputs to `dist/` |
| **Docker build** | Multi-stage build — Node for compiling, Nginx for serving |
| **Push** | Pushes the image to the local homelab Docker registry |
| **Deploy** | `kubectl set image` to roll out the new image to k3s |

Pull requests run lint + build only. Full deploy only triggers on merge to `main`.

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite |
| AI | Anthropic Claude API (claude-sonnet-4) |
| Data | GitHub REST API |
| Container | Docker (multi-stage, Nginx alpine) |
| Orchestration | k3s (Kubernetes) |
| CI/CD | GitHub Actions + self-hosted runner |
| Ingress | Nginx ingress controller |
| OS | Ubuntu 24.04 |

---

## Project structure

```
github-contribution-finder/
├── .github/
│   └── workflows/
│       └── ci-cd.yml        # Full lint → build → push → deploy pipeline
├── k8s/
│   ├── deployment.yaml      # 2 replicas, resource limits, liveness probe
│   ├── service.yaml         # ClusterIP service
│   └── ingress.yaml         # Nginx ingress with host routing
├── src/
│   ├── App.jsx              # Main React component
│   └── main.jsx             # Entry point
├── Dockerfile               # Multi-stage: node:alpine → nginx:alpine
├── nginx.conf               # Nginx config for serving React SPA
├── .eslintrc.json           # ESLint rules
└── README.md
```

---

## Running locally

**Prerequisites:** Node 18+, an Anthropic API key

```bash
git clone https://github.com/YOUR_USERNAME/github-contribution-finder
cd github-contribution-finder

npm install

# Add your Anthropic API key
echo "VITE_ANTHROPIC_API_KEY=sk-ant-..." > .env.local

npm run dev
# → http://localhost:5173
```

**Running with Docker:**

```bash
docker build -t github-contribution-finder .
docker run -p 8080:80 github-contribution-finder
# → http://localhost:8080
```

---

## Deploying to k3s

**1. Register the self-hosted GitHub Actions runner** on your homelab machine:

```bash
# On your homelab — follow the setup instructions at:
# GitHub repo → Settings → Actions → Runners → New self-hosted runner
```

**2. Apply the Kubernetes manifests:**

```bash
kubectl apply -f k8s/
```

**3. Add your Anthropic API key as a Kubernetes secret:**

```bash
kubectl create secret generic app-secrets \
  --from-literal=ANTHROPIC_API_KEY=sk-ant-...
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
| `ANTHROPIC_API_KEY` | k8s secret | Anthropic API key for Claude |
| `REGISTRY` | Actions secret | Local registry URL e.g. `registry.homelab:5000` |
| `KUBE_CONFIG` | Actions secret | Base64-encoded kubeconfig |

---

## License

MIT