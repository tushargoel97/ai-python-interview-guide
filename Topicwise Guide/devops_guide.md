## Table of Contents

1. [1. Docker, Kubernetes & Container Orchestration](#1-docker-kubernetes--container-orchestration)
2. [2. CI/CD Pipelines](#2-cicd-pipelines)

### 1. Docker, Kubernetes & Container Orchestration

> **📣 Definition:** _"Docker packages an application + its dependencies into a portable container that runs identically anywhere. Kubernetes orchestrates fleets of containers — handling scaling (HPA), self-healing (replacing crashed pods), service discovery, rolling deployments, and resource scheduling. Together they're the de-facto standard for deploying modern microservices, with Pods (groups of containers) as the unit of deployment, Services as stable network endpoints, and Deployments as the spec for what should be running."_

**Layman**: Docker = a standardized shipping container for your app (runs the same everywhere). Kubernetes = the port authority that decides how many containers to run, where to put them, and replaces broken ones.

**Technical key points**: Pods, Services, Deployments, ReplicaSets, ConfigMaps, Secrets, Liveness/Readiness probes, HPA (Horizontal Pod Autoscaler), Ingress controllers, Service Mesh (Istio).

**📌 TLDR**: "Docker packages apps into portable containers. Kubernetes orchestrates them — handles scaling, self-healing, rolling deployments, and service discovery. Know Pods, Deployments, Services, HPA, and health probes."

---

### 2. CI/CD Pipelines

> **📣 Definition:** _"Continuous Integration/Continuous Deployment is the practice of automatically building, testing, and deploying code changes. CI runs tests + linting + security scans on every commit. CD ships the artifact through staging to production with controlled rollout (canary, blue-green) so failures are caught before full impact. Tools: GitHub Actions, GitLab CI, Jenkins, ArgoCD. Combine with Infrastructure as Code (Terraform) for reproducible deployments. The goal is to make deploying boring — fast, predictable, and reversible."_

**Layman**: An assembly line for code. Every time you push code, robots automatically test it, check for bugs, and if everything passes, deploy it to production — no manual work.

**Technical**: Source → Build → Test (unit, integration, e2e) → Static Analysis (linting, type checking, security scanning) → Container Build → Deploy (staging → production with canary/blue-green). Tools: GitHub Actions, GitLab CI, Jenkins, ArgoCD.

**📌 TLDR**: "CI/CD automates testing and deployment. CI = build + test on every commit. CD = automated deployment to staging/production. Blue-green and canary deployments minimize risk. Infrastructure as Code (Terraform) for reproducibility."

---
