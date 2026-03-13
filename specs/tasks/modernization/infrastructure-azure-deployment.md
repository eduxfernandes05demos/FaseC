# Task: Infrastructure — Azure Deployment

> Azure deployment: AKS/ACA, networking, storage, monitoring, Bicep/Terraform IaC.

---

## Metadata

| Field | Value |
|-------|-------|
| **Priority** | P2 — Cloud-Native |
| **Estimated Complexity** | Medium |
| **Estimated Effort** | 1 week (infrastructure) + ongoing operations |
| **Dependencies** | Container system, Session manager, Streaming gateway |
| **Blocks** | Production deployment |
| **Phase** | Phase 4 — Cloud-Native |

---

## Objective

Deploy the complete Quake cloud-streaming stack on Azure using Infrastructure as Code (Bicep), with AKS for container orchestration, ACR for image management, Azure Monitor for observability, and proper networking for both UDP game traffic and WebSocket streaming.

---

## Azure Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Azure Subscription                            │
│                                                                      │
│  ┌─────────────┐   ┌──────────────────────────────────────────────┐  │
│  │ Azure DNS   │   │              Virtual Network (10.0.0.0/16)   │  │
│  │ quake.app   │   │                                              │  │
│  └──────┬──────┘   │  ┌───────────────────────────────────────┐   │  │
│         │          │  │  AKS Cluster (quake-aks)              │   │  │
│         │          │  │                                       │   │  │
│  ┌──────▼──────┐   │  │  ┌─────────────────────────────────┐ │   │  │
│  │ App Gateway │───│──│──│ Ingress Controller               │ │   │  │
│  │ (WAF v2)    │   │  │  │  HTTPS → Gateway Service         │ │   │  │
│  └─────────────┘   │  │  │  UDP   → Engine Service           │ │   │  │
│                    │  │  └─────────────────────────────────┘ │   │  │
│                    │  │                                       │   │  │
│                    │  │  ┌──────────┐ ┌──────────┐           │   │  │
│                    │  │  │ Gateway  │ │ Session  │           │   │  │
│                    │  │  │ Pod(s)   │ │ Manager  │           │   │  │
│                    │  │  └──────────┘ └──────────┘           │   │  │
│                    │  │                                       │   │  │
│                    │  │  ┌──────────┐ ┌──────────┐ ┌──────┐ │   │  │
│                    │  │  │Engine    │ │Engine    │ │Eng...│ │   │  │
│                    │  │  │Session 1 │ │Session 2 │ │      │ │   │  │
│                    │  │  └──────────┘ └──────────┘ └──────┘ │   │  │
│                    │  └───────────────────────────────────────┘   │  │
│                    │                                              │  │
│  ┌─────────────┐  │  ┌──────────────┐  ┌───────────────────────┐ │  │
│  │ ACR         │  │  │ Key Vault    │  │ Log Analytics + Monitor│ │  │
│  │ (Container  │  │  │ (Secrets)    │  │ (Observability)        │ │  │
│  │  Registry)  │  │  └──────────────┘  └───────────────────────┘ │  │
│  └─────────────┘  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Infrastructure as Code (Bicep)

### Directory Structure

```
infrastructure/
├── main.bicep                  # Orchestration
├── modules/
│   ├── aks.bicep               # AKS cluster
│   ├── acr.bicep               # Container registry
│   ├── vnet.bicep              # Virtual network
│   ├── keyvault.bicep          # Secrets management
│   ├── monitor.bicep           # Azure Monitor + Log Analytics
│   ├── appgw.bicep             # Application Gateway
│   └── dns.bicep               # DNS zone
├── parameters/
│   ├── dev.bicepparam          # Development
│   ├── staging.bicepparam      # Staging
│   └── prod.bicepparam         # Production
└── deploy.sh                   # Deployment script
```

### Main Orchestration (`main.bicep`)

```bicep
targetScope = 'subscription'

param environment string = 'dev'
param location string = 'westeurope'

resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name: 'quake-${environment}-rg'
  location: location
}

module vnet 'modules/vnet.bicep' = {
  scope: rg
  name: 'vnet'
  params: { location: location, environment: environment }
}

module acr 'modules/acr.bicep' = {
  scope: rg
  name: 'acr'
  params: { location: location, environment: environment }
}

module aks 'modules/aks.bicep' = {
  scope: rg
  name: 'aks'
  params: {
    location: location
    environment: environment
    vnetSubnetId: vnet.outputs.aksSubnetId
    acrId: acr.outputs.acrId
  }
}

module monitor 'modules/monitor.bicep' = {
  scope: rg
  name: 'monitor'
  params: { location: location, environment: environment }
}

module keyvault 'modules/keyvault.bicep' = {
  scope: rg
  name: 'keyvault'
  params: { location: location, environment: environment }
}
```

### AKS Module (`modules/aks.bicep`)

```bicep
param location string
param environment string
param vnetSubnetId string
param acrId string

var aksName = 'quake-${environment}-aks'

resource aks 'Microsoft.ContainerService/managedClusters@2023-10-01' = {
  name: aksName
  location: location
  identity: { type: 'SystemAssigned' }
  properties: {
    dnsPrefix: 'quake-${environment}'
    agentPoolProfiles: [
      {
        name: 'system'
        count: environment == 'prod' ? 3 : 1
        vmSize: environment == 'prod' ? 'Standard_D4s_v3' : 'Standard_B2s'
        vnetSubnetID: vnetSubnetId
        mode: 'System'
        enableAutoScaling: true
        minCount: 1
        maxCount: environment == 'prod' ? 10 : 3
      }
    ]
    networkProfile: {
      networkPlugin: 'azure'
      loadBalancerSku: 'standard'
    }
    addonProfiles: {
      omsagent: {
        enabled: true
      }
    }
  }
}
```

---

## Kubernetes Manifests

### Engine Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quake-engine
  namespace: quake
spec:
  replicas: 1  # Managed by session manager
  selector:
    matchLabels:
      app: quake-engine
  template:
    metadata:
      labels:
        app: quake-engine
    spec:
      containers:
      - name: quake
        image: quakeacr.azurecr.io/quake/engine:latest
        ports:
        - containerPort: 26000
          protocol: UDP
        - containerPort: 8080
          protocol: TCP
        env:
        - name: QUAKE_PORT
          value: "26000"
        - name: QUAKE_GAMEDIR
          value: "id1"
        - name: QUAKE_MEMORY
          value: "32"
        - name: LOG_FORMAT
          value: "json"
        resources:
          requests:
            memory: "64Mi"
            cpu: "500m"
          limits:
            memory: "128Mi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 5
        securityContext:
          runAsUser: 1000
          runAsNonRoot: true
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
      volumes:
      - name: game-data
        persistentVolumeClaim:
          claimName: quake-game-data
```

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: quake-engine-hpa
  namespace: quake
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: quake-engine
  minReplicas: 1
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: quake-engine-policy
  namespace: quake
spec:
  podSelector:
    matchLabels:
      app: quake-engine
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: quake-gateway
    ports:
    - port: 8080     # Health checks
      protocol: TCP
  - ports:
    - port: 26000     # Game traffic (from external)
      protocol: UDP
  egress:
  - {}  # Allow all egress
```

---

## Deployment Process

```bash
# 1. Deploy Azure infrastructure
az deployment sub create \
  --location westeurope \
  --template-file infrastructure/main.bicep \
  --parameters infrastructure/parameters/dev.bicepparam

# 2. Get AKS credentials
az aks get-credentials -g quake-dev-rg -n quake-dev-aks

# 3. Build and push container images
az acr build -r quakedevacr -t quake/engine:latest legacy-src/
az acr build -r quakedevacr -t quake/gateway:latest services/streaming-gateway/
az acr build -r quakedevacr -t quake/session-mgr:latest services/session-manager/

# 4. Deploy Kubernetes resources
kubectl apply -f k8s/base/
kubectl apply -f k8s/dev/  # or staging/ or production/

# 5. Verify deployment
kubectl get pods -n quake
kubectl get svc -n quake
```

---

## Monitoring Setup

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| Pod restarts | Kubernetes | > 3 in 10 min |
| CPU usage | Container | > 90% sustained |
| Memory usage | Container | > 90% of limit |
| Error rate | Application logs | > 5% in 5 min |
| Active sessions | Session Manager | Approaching max |
| Network throughput | AKS | > 80% capacity |

---

## Files to Create

| File | Description |
|------|-------------|
| `infrastructure/main.bicep` | Main orchestration |
| `infrastructure/modules/aks.bicep` | AKS cluster definition |
| `infrastructure/modules/acr.bicep` | Container registry |
| `infrastructure/modules/vnet.bicep` | Network configuration |
| `infrastructure/modules/keyvault.bicep` | Secrets management |
| `infrastructure/modules/monitor.bicep` | Monitoring setup |
| `infrastructure/parameters/dev.bicepparam` | Dev environment params |
| `infrastructure/parameters/prod.bicepparam` | Prod environment params |
| `infrastructure/deploy.sh` | Deployment automation script |
| `k8s/base/namespace.yaml` | Kubernetes namespace |
| `k8s/base/engine-deployment.yaml` | Engine pod spec |
| `k8s/base/gateway-deployment.yaml` | Gateway pod spec |
| `k8s/base/session-mgr-deployment.yaml` | Session manager pod spec |
| `k8s/base/network-policies.yaml` | Network restrictions |
| `k8s/base/hpa.yaml` | Auto-scaling configuration |

---

## Acceptance Criteria

- [ ] `az deployment sub create` provisions all Azure resources
- [ ] AKS cluster is running and accessible via `kubectl`
- [ ] Container images push to ACR successfully
- [ ] All pods start and pass health checks
- [ ] UDP game traffic reaches engine pods from external clients
- [ ] WebSocket streaming traffic reaches gateway pods
- [ ] Azure Monitor receives logs and metrics
- [ ] Horizontal Pod Autoscaler scales engine pods under load
- [ ] Network policies restrict inter-pod communication correctly
- [ ] Deployment is repeatable (idempotent Bicep templates)
- [ ] Cost is within budget (< $500/month for dev environment)
