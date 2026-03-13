# DevOps Transformation Plan — Quake Engine

> DevOps improvements: CI/CD pipelines, container builds, automated testing, Azure deployment automation.
> All references point to actual source files in this repository.

---

## 1. Executive Summary

This document outlines the DevOps transformation from manual, platform-specific builds to a fully automated CI/CD pipeline with container builds, automated testing, security scanning, and Azure deployment automation using Infrastructure as Code.

---

## 2. Current State

| Capability | Status | Details |
|------------|--------|---------|
| Version control | ✅ Git | GitHub repository |
| Build automation | ❌ Manual | Platform-specific Makefiles + MSVC projects |
| CI/CD pipeline | ❌ None | No GitHub Actions or other CI system |
| Testing | ❌ None | Zero tests in codebase |
| Container builds | ❌ None | No Dockerfile |
| Infrastructure as Code | ❌ None | No Terraform/Bicep |
| Monitoring | ❌ None | No observability infrastructure |
| Release management | ❌ None | No versioning or release automation |

---

## 3. CI/CD Pipeline Design

### 3.1 Pipeline Architecture

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Commit   │───▶│  Build   │───▶│   Test   │───▶│ Security │───▶│  Deploy  │
│           │    │          │    │          │    │   Scan   │    │          │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
     │               │               │               │               │
     │          CMake build      Unit tests       CodeQL          Staging
     │          Multi-platform   Integration     Trivy scan       Production
     │          Container build  Benchmarks      SBOM gen         AKS deploy
```

### 3.2 GitHub Actions Workflows

#### Build & Test Workflow (`.github/workflows/ci.yml`)

```yaml
name: CI
on: [push, pull_request]
jobs:
  build-linux:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install dependencies
        run: sudo apt-get install -y cmake gcc libsdl2-dev
      - name: Configure
        run: cmake -B build -DCMAKE_BUILD_TYPE=Release -DHEADLESS=ON
      - name: Build
        run: cmake --build build -j$(nproc)
      - name: Test
        run: cd build && ctest --output-on-failure

  build-container:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build container
        run: docker build -t quake-engine:${{ github.sha }} .
      - name: Container test
        run: docker run --rm quake-engine:${{ github.sha }} --selftest
```

#### Security Scan Workflow (`.github/workflows/security.yml`)

```yaml
name: Security
on: [push, pull_request]
jobs:
  codeql:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with: { languages: c-cpp }
      - name: Build
        run: cmake -B build && cmake --build build
      - uses: github/codeql-action/analyze@v3

  container-scan:
    runs-on: ubuntu-latest
    needs: build-container
    steps:
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: quake-engine:latest
          severity: CRITICAL,HIGH
```

#### Deploy Workflow (`.github/workflows/deploy.yml`)

```yaml
name: Deploy
on:
  push:
    tags: ['v*']
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: azure/login@v1
      - name: Deploy to AKS staging
        run: |
          az aks get-credentials -g quake-rg -n quake-aks
          kubectl apply -f k8s/staging/

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production
    steps:
      - uses: azure/login@v1
      - name: Deploy to AKS production
        run: |
          az aks get-credentials -g quake-rg -n quake-aks
          kubectl apply -f k8s/production/
```

---

## 4. Container Strategy

### 4.1 Image Hierarchy

```
ubuntu:22.04 (base)
    │
    ├── quake-builder:latest    (build tools + CMake)
    │
    ├── quake-engine:latest     (headless engine runtime)
    │
    ├── quake-server:latest     (dedicated server)
    │
    ├── quake-gateway:latest    (streaming gateway)
    │
    └── quake-session-mgr:latest (session manager)
```

### 4.2 Container Registry

| Component | Registry | Tagging |
|-----------|----------|---------|
| All images | Azure Container Registry (ACR) | `<acr>.azurecr.io/quake/<image>:<tag>` |
| Tag format | SemVer + SHA | `v1.0.0`, `v1.0.0-abc1234`, `latest` |
| Retention | 90 days for non-release tags | ACR retention policy |

### 4.3 Multi-Stage Dockerfile

```dockerfile
# Stage 1: Build
FROM ubuntu:22.04 AS builder
RUN apt-get update && apt-get install -y \
    gcc cmake make libsdl2-dev \
    && rm -rf /var/lib/apt/lists/*
COPY legacy-src/ /src/legacy-src/
WORKDIR /src/Quake
RUN cmake -B build \
    -DCMAKE_BUILD_TYPE=Release \
    -DHEADLESS=ON \
    -DUSE_ASM=OFF \
    && cmake --build build -j$(nproc)

# Stage 2: Runtime
FROM ubuntu:22.04 AS runtime
RUN apt-get update && apt-get install -y \
    libsdl2-2.0-0 curl \
    && rm -rf /var/lib/apt/lists/* \
    && useradd -m -s /bin/bash quake
COPY --from=builder /src/legacy-src/build/quake-headless /usr/local/bin/
COPY --from=builder /src/legacy-src/build/quake-server /usr/local/bin/
USER quake
WORKDIR /home/quake
EXPOSE 26000/udp 8080/tcp
HEALTHCHECK --interval=10s --timeout=3s \
    CMD curl -f http://localhost:8080/healthz || exit 1
ENTRYPOINT ["quake-headless"]
```

---

## 5. Testing Automation

### 5.1 Test Pyramid

```
        ┌─────────┐
        │  E2E    │  3-5 tests  — Full streaming pipeline
        ├─────────┤
        │ Integr. │  20-30 tests — Subsystem interactions
        ├─────────┤
        │  Unit   │  100+ tests — Individual functions
        └─────────┘
```

### 5.2 Test Framework

| Level | Framework | Runner |
|-------|-----------|--------|
| Unit | Unity (C) or Check | CTest |
| Integration | Custom harness | CTest + Docker |
| E2E | pytest + WebSocket client | GitHub Actions |
| Performance | Custom benchmark | CTest + reporting |

### 5.3 Test Categories

| Category | Tests | Source |
|----------|-------|--------|
| Memory safety | Buffer overflow checks | `snprintf` return value tests |
| Math correctness | Vector/matrix operations | `mathlib.c` functions |
| Network protocol | Packet encoding/decoding | `net_dgrm.c`, `sv_ents.c` |
| File I/O | Path sanitization | `common.c` file functions |
| QuakeC VM | Opcode execution | `pr_exec.c` |
| Configuration | Cvar parsing | `cvar.c` |
| Memory allocator | Zone/Hunk operations | `zone.c` |

---

## 6. Infrastructure as Code

### 6.1 Azure Resources (Bicep)

```
quake-infrastructure/
├── main.bicep              # Orchestration
├── modules/
│   ├── aks.bicep           # Azure Kubernetes Service
│   ├── acr.bicep           # Azure Container Registry
│   ├── vnet.bicep          # Virtual Network
│   ├── keyvault.bicep      # Secrets management
│   ├── monitor.bicep       # Azure Monitor + Log Analytics
│   └── dns.bicep           # Azure DNS zone
├── parameters/
│   ├── dev.bicepparam      # Development environment
│   ├── staging.bicepparam  # Staging environment
│   └── prod.bicepparam     # Production environment
└── scripts/
    ├── deploy.sh           # Deployment script
    └── teardown.sh         # Cleanup script
```

### 6.2 Azure Resource Map

| Resource | Purpose | SKU |
|----------|---------|-----|
| AKS Cluster | Container orchestration | Standard_D4s_v3 (dev: Standard_B2s) |
| ACR | Container images | Basic (dev) / Standard (prod) |
| VNet + Subnet | Network isolation | /16 with /24 subnets |
| Key Vault | Secrets management | Standard |
| Log Analytics | Centralized logging | Per-GB pricing |
| Azure Monitor | Metrics and alerts | Standard |
| Azure DNS | Public DNS | Standard |
| Application Gateway | Ingress controller | Standard_v2 |

### 6.3 Kubernetes Manifests

```
k8s/
├── base/
│   ├── namespace.yaml
│   ├── engine-deployment.yaml
│   ├── engine-service.yaml
│   ├── engine-hpa.yaml          # Horizontal Pod Autoscaler
│   ├── gateway-deployment.yaml
│   ├── gateway-service.yaml
│   ├── session-mgr-deployment.yaml
│   ├── session-mgr-service.yaml
│   ├── configmap.yaml
│   └── network-policies.yaml
├── staging/
│   └── kustomization.yaml
└── production/
    └── kustomization.yaml
```

---

## 7. Monitoring & Observability

### 7.1 Metrics

| Metric | Source | Dashboard |
|--------|--------|-----------|
| Frame render time | Engine | Grafana |
| Sessions active | Session Manager | Azure Monitor |
| Stream latency | Gateway | Grafana |
| Container CPU/memory | Kubernetes | Azure Monitor |
| Error rate | Structured logs | Azure Monitor |
| Network throughput | Engine + Gateway | Grafana |

### 7.2 Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| High error rate | > 5% errors in 5 min | Critical |
| Pod crash loop | > 3 restarts in 10 min | Critical |
| High latency | > 100ms frame latency | Warning |
| Memory pressure | > 90% memory usage | Warning |
| Scaling limit | Max replicas reached | Warning |

### 7.3 Logging

| Log Source | Format | Destination |
|-----------|--------|-------------|
| Engine | JSON (structured) | stdout → Azure Log Analytics |
| Gateway | JSON (structured) | stdout → Azure Log Analytics |
| Session Manager | JSON (structured) | stdout → Azure Log Analytics |
| Kubernetes | Native | Azure Log Analytics |

---

## 8. Release Management

### 8.1 Versioning

| Component | Format | Example |
|-----------|--------|---------|
| Engine | SemVer | `v1.0.0` |
| Container images | SemVer + SHA | `v1.0.0-abc1234` |
| Infrastructure | SemVer | `v1.0.0` |
| Kubernetes manifests | SemVer | `v1.0.0` |

### 8.2 Release Process

```
1. Create release branch: release/v1.x
2. Run full CI pipeline (build + test + security)
3. Generate changelog from conventional commits
4. Build and tag container images
5. Deploy to staging environment
6. Run E2E tests against staging
7. Manual approval for production
8. Deploy to production (rolling update)
9. Monitor for 24 hours
10. Tag Git release
```

---

## 9. Implementation Timeline

| Week | Deliverable | Dependency |
|------|-------------|-----------|
| 1 | GitHub Actions CI workflow (build) | CMake build system |
| 2 | Test framework + first tests | CI workflow |
| 3 | Security scanning (CodeQL + Trivy) | CI workflow |
| 4 | Dockerfile + container build pipeline | CI workflow |
| 5 | Azure infrastructure (Bicep) | Container builds |
| 6 | Kubernetes manifests + deploy pipeline | Azure infra |
| 7 | Monitoring and alerting setup | Azure infra |
| 8 | Release automation + documentation | All above |
