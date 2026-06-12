# CrowdStrike Falcon KAC + IAR Quick Start Deployment Guide
## Kubernetes Admission Controller & Image Assessment at Runtime (Watcher Mode)

---

## Overview

This guide deploys two complementary CrowdStrike Falcon components to a Kubernetes cluster:

| Component | What It Does | Workload Type |
|-----------|-------------|---------------|
| **Falcon KAC** (Kubernetes Admission Controller) | Validates workloads at admission time, detects misconfigurations (IOMs), and can block non-compliant pods from being scheduled | Deployment (2 containers: webhook + controller) |
| **Falcon IAR** (Image Assessment at Runtime) | Scans container images running in the cluster and reports vulnerabilities to the Falcon console | Deployment — Watcher Mode (1 replica) |

**Why Watcher Mode for IAR?**
Watcher mode runs IAR as a single Deployment (not a DaemonSet). It discovers running pods via the Kubernetes API and pulls images for scanning through the registry — no container runtime socket access required. This means:
- No privileged DaemonSet on every node
- Lower operational overhead and blast radius
- Works on managed clusters (EKS Fargate, GKE Autopilot) where node-level socket access is restricted
- Simpler RBAC — no hostPath mounts

> Use DaemonSet (socket) mode only if you need local runtime access or are in an air-gapped environment where registry pull from the IAR pod is not possible.

---

## Prerequisites

| Requirement | Details |
|------------|---------|
| Helm | 3.x installed and in PATH |
| kubectl | Configured and authenticated to your target cluster |
| Cluster admin | Required for RBAC, namespace creation, and webhook registration |
| CrowdStrike CID | Customer ID with checksum (e.g., `ABCDEF1234-56`) |
| Falcon API credentials | OAuth2 Client ID + Secret with **Falcon Container Image** scope (for IAR) |
| Image access | Either CrowdStrike registry token (`dockerAPIToken`) or images mirrored to a private registry |
| Architecture | x86_64 or ARM64 (or mixed) nodes |

### Supported Distributions
EKS, EKS Fargate, AKS, GKE, GKE Autopilot, K3s

### Required Outbound Network Access

**KAC** — allow TLS/443 to your cloud's sink addresses:

| Cloud | Addresses |
|-------|-----------|
| US-1 | `ts01-b.cloudsink.net`, `lfodown01-b.cloudsink.net` |
| US-2 | `ts01-gyr-maverick.cloudsink.net`, `lfodown01-gyr-maverick.cloudsink.net` |
| EU-1 | `ts01-lanner-lion.cloudsink.net`, `lfodown01-lanner-lion.cloudsink.net` |
| US-GOV-1 | `ts01-laggar-gcw.cloudsink.net`, `lfodown01-laggar-gcw.cloudsink.net` |

**IAR** — allow HTTPS to your region's API and upload servers:

| Region | Auth API | Primary Upload | Secondary Upload |
|--------|----------|---------------|-----------------|
| US-1 | `api.crowdstrike.com` | `container-upload.us-1.crowdstrike.com` | `api.crowdstrike.com` |
| US-2 | `api.us-2.crowdstrike.com` | `container-upload.us-2.crowdstrike.com` | `api.us-2.crowdstrike.com` |
| EU-1 | `api.eu-1.crowdstrike.com` | `container-upload.eu-1.crowdstrike.com` | `api.eu-1.crowdstrike.com` |
| US-GOV-1 | `api.laggar.gcw.crowdstrike.com` | `container-upload.laggar.gcw.crowdstrike.com` | `api.laggar.gcw.crowdstrike.com` |
| US-GOV-2 | `api.us-gov-2.crowdstrike.mil` | `container-upload.us-gov-2.crowdstrike.mil` | `api.us-gov-2.crowdstrike.mil` |

---

## Step 1: Add the CrowdStrike Helm Repository

```bash
helm repo add crowdstrike https://crowdstrike.github.io/falcon-helm
helm repo update
```

---

## Step 2: Set Environment Variables

Set these once — they will be referenced throughout the guide.

```bash
# CrowdStrike CID (with checksum — found on the Sensor Downloads page)
export FALCON_CID="YOUR_CID_WITH_CHECKSUM"

# Your CrowdStrike cloud region: us-1 | us-2 | eu-1 | gov1 | gov2
export FALCON_REGION="us-1"

# Falcon API OAuth2 credentials (required for IAR)
export FALCON_CLIENT_ID="your_oauth_client_id"
export FALCON_CLIENT_SECRET="your_oauth_client_secret"

# KAC image (from CrowdStrike registry or your mirror)
# Tag format: <version>.container.x86_64.Release.<REGION>
# Example: 7.33.0-1.container.x86_64.Release.US-1
export KAC_IMAGE_REPO="registry.crowdstrike.com/falcon-kac/release/falcon-kac"
export KAC_IMAGE_TAG="7.33.0-1.container.x86_64.Release.US-1"

# IAR image — region-specific registry path
# US-1/US-2/EU-1: registry.crowdstrike.com
# GOV-1:          registry.laggar.gcw.crowdstrike.com
# GOV-2:          registry.us-gov-2.crowdstrike.mil
export IAR_IMAGE_REPO="registry.crowdstrike.com/falcon-imageanalyzer/release/falcon-imageanalyzer"
export IAR_IMAGE_TAG="1.0.24"       # Use the latest available version

# (Optional) If pulling images directly from CrowdStrike registry
export CS_REGISTRY_TOKEN="your_crowdstrike_artifactory_token"
```

> **Tip:** To find your CID with checksum, go to **Falcon Console → Sensor Downloads** and click
> *Copy your Customer ID checksum to the clipboard*.

> **Tip:** To create an API client, go to **Falcon Console → Support & Resources → API Clients and Keys**.
> The client requires the **Falcon Container Image** scope (Read + Write).

---

## Step 3: Apply Pod Security Standards (Kubernetes 1.25+)

Both namespaces require `privileged` Pod Security Standards labels.
IAR in watcher mode runs as root (UID 0) for image scanning.
KAC requires privileged namespace access for the webhook containers.

```bash
# KAC namespace
kubectl create namespace falcon-kac --dry-run=client -o yaml | kubectl apply -f -
kubectl label --overwrite ns falcon-kac \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged \
  pod-security.kubernetes.io/audit=privileged

# IAR namespace
kubectl create namespace falcon-image-analyzer --dry-run=client -o yaml | kubectl apply -f -
kubectl label --overwrite ns falcon-image-analyzer \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged \
  pod-security.kubernetes.io/audit=privileged
```

---

## Step 4: Deploy Falcon KAC

### 4a. Basic Install (Public Registry)

```bash
helm upgrade --install falcon-kac crowdstrike/falcon-kac \
  -n falcon-kac --create-namespace \
  --set falcon.cid="${FALCON_CID}" \
  --set image.repository="${KAC_IMAGE_REPO}" \
  --set image.tag="${KAC_IMAGE_TAG}"
```

### 4b. Install with Private Registry Authentication

If your registry requires credentials, encode your Docker config and pass it as a pull secret:

```bash
export KAC_PULL_TOKEN=$(cat ~/.docker/config.json | base64 -w 0)

helm upgrade --install falcon-kac crowdstrike/falcon-kac \
  -n falcon-kac --create-namespace \
  --set falcon.cid="${FALCON_CID}" \
  --set image.repository="${KAC_IMAGE_REPO}" \
  --set image.tag="${KAC_IMAGE_TAG}" \
  --set image.registryConfigJSON="${KAC_PULL_TOKEN}"
```

### 4c. Verify KAC Deployment

```bash
# Check pod status — you should see 1 Running pod with 2 containers
kubectl get pods -n falcon-kac

# Confirm running container image versions
kubectl get pods -n falcon-kac \
  -o jsonpath='{range .items[*]}{"\n"}{.metadata.name}{":\t"}{range .spec.containers[*]}{.image}{", "}{end}{end}'

# Check KAC logs for connectivity
kubectl logs -n falcon-kac -l app.kubernetes.io/name=falcon-kac -c falcon-ac
```

**Expected:** One pod in `Running` state with 2 ready containers (`falcon-client` webhook and `falcon-ac` controller).

---

## Step 5: Deploy Falcon IAR (Watcher Mode)

Create a values file to keep credentials out of shell history and simplify upgrades.

### 5a. Create `iar-values.yaml`

```yaml
# IAR Watcher Mode — single Deployment, no node-level socket access required
deployment:
  enabled: true
daemonset:
  enabled: false   # Must be false when deployment is true

image:
  repository: "registry.crowdstrike.com/falcon-imageanalyzer/release/falcon-imageanalyzer"
  tag: "1.0.24"    # Pin to a specific version; avoid 'latest'

crowdstrikeConfig:
  cid: "YOUR_CID_WITH_CHECKSUM"
  clientID: "your_oauth_client_id"
  clientSecret: "your_oauth_client_secret"
  agentRegion: "us-1"             # us-1 | us-2 | eu-1 | gov1 | gov2
  clusterName: ""                 # Leave blank for auto-discovery (EKS/AKS/GKE)
                                  # Set explicitly for on-prem or MicroK8s

# Private registry credential discovery (recommended)
privateRegistries:
  autoDiscoverCredentials: true   # IAR auto-discovers docker-registry secrets cluster-wide

# Temp working space for image layer extraction
# Rule of thumb: 2x the size of your largest container image
volumes:
  - name: tmp-volume
    emptyDir:
      sizeLimit: 20Gi

# Prevent pod eviction under node memory/ephemeral pressure
priorityClassName: "system-cluster-critical"

# Exclude system namespaces from scanning (recommended)
exclusions:
  namespace: "kube-system,kube-public,falcon-kac,falcon-image-analyzer"
```

> **Security note:** For production, store `clientID`, `clientSecret`, and `cid` in a Kubernetes
> Secret and reference it via `crowdstrikeConfig.existingSecret` instead of plaintext values.

### 5b. Using an Existing Kubernetes Secret (Recommended for Production)

Create the secret manually:

```bash
kubectl create secret generic falcon-iar-credentials \
  -n falcon-image-analyzer \
  --from-literal=FALCON_CLIENT_ID="${FALCON_CLIENT_ID}" \
  --from-literal=FALCON_CLIENT_SECRET="${FALCON_CLIENT_SECRET}" \
  --from-literal=FALCON_CID="${FALCON_CID}"
```

Then replace the `crowdstrikeConfig` block in `iar-values.yaml` with:

```yaml
crowdstrikeConfig:
  agentRegion: "us-1"
  clusterName: ""
  existingSecret: "falcon-iar-credentials"
```

### 5c. Install IAR

```bash
helm upgrade --install imageanalyzer crowdstrike/falcon-image-analyzer \
  -n falcon-image-analyzer --create-namespace \
  -f iar-values.yaml
```

### 5d. Verify IAR Deployment

```bash
# Check pod status — expect 1 Running pod (Deployment, 1 replica)
kubectl get pods -n falcon-image-analyzer

# Confirm the pod is running in watcher (Deployment) mode
kubectl get deployment -n falcon-image-analyzer

# Check IAR logs for startup and scan activity
kubectl logs -n falcon-image-analyzer \
  -l app.kubernetes.io/name=falcon-image-analyzer --tail=50
```

**Expected:** One pod in `Running` state. Logs should show cluster connection, pod discovery, and image scan submissions.

---

## Step 6: Verify KAC + IAR Integration

KAC and IAR communicate directly — KAC can request image scan data from IAR for admission decisions.
This requires IAR to be in the `falcon-image-analyzer` namespace (the default) and KAC >= 1.5.2 / IAR >= 1.0.17.

```bash
# Confirm KAC can see the IAR namespace
kubectl get svc -n falcon-image-analyzer

# Confirm KAC knows the IAR namespace (check KAC configmap)
kubectl get configmap -n falcon-kac -o yaml
```

If you deployed IAR to a non-default namespace, tell KAC where it is:

```bash
helm upgrade falcon-kac crowdstrike/falcon-kac \
  -n falcon-kac \
  --reuse-values \
  --set falconImageAnalyzerNamespace="your-custom-namespace"
```

---

## Step 7: Validate End-to-End

Deploy a test workload and confirm both KAC and IAR are active:

```bash
# Deploy a simple test pod
kubectl run test-nginx --image=nginx:latest -n default

# KAC: Check the Falcon console → Kubernetes Admission Control → Events
# for the new admission event

# IAR: Check the Falcon console → Image Assessment → Images
# for the nginx image scan result (may take a few minutes)

# Check KAC admission webhook is registered
kubectl get validatingwebhookconfigurations | grep falcon
```

---

## Optional Configuration

### Sensor Tags (for grouping in Falcon console)

```bash
helm upgrade falcon-kac crowdstrike/falcon-kac -n falcon-kac \
  --reuse-values \
  --set falcon.tags="env:production,team:platform"
```

### Proxy Support (IAR)

Add to `iar-values.yaml`:

```yaml
proxyConfig:
  HTTP_PROXY: "http://proxy.example.com:3128"
  HTTPS_PROXY: "http://proxy.example.com:3128"
  NO_PROXY: "registry.crowdstrike.com,your-private-registry.example.com,10.0.0.0/8"
```

> **Important:** Add all registry domains to `NO_PROXY` so IAR can pull images directly without proxy interference.

### AWS EKS — ECR Access via IRSA

Add to `iar-values.yaml` for OIDC-based ECR authentication:

```yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::111122223333:role/falcon-iar-ecr-role"
```

The IAM role must allow `ecr:*` on the relevant registries and have a trust relationship with the
`falcon-image-analyzer` service account in the `falcon-image-analyzer` namespace.

### GKE Autopilot — PVC for Temp Volume

GKE Autopilot limits `emptyDir` to 10Gi. If your images exceed that, replace the temp volume with a PVC:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: iar-tmp-pvc
  namespace: falcon-image-analyzer
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: premium-rwo
  resources:
    requests:
      storage: 20Gi
EOF
```

Then in `iar-values.yaml`:

```yaml
volumes:
  - name: tmp-volume
    persistentVolumeClaim:
      claimName: iar-tmp-pvc
volumeMounts:
  - mountPath: /tmp
    name: tmp-volume
```

### Excluding Workloads from IAR Scanning

**By namespace (values file):**
```yaml
exclusions:
  namespace: "dev,staging,test"
```

**By annotation (applied to any running namespace):**
```bash
kubectl annotate namespace my-namespace sensor.crowdstrike.com/imageanalyzer=disabled
```

**By annotation (applied to a specific pod/deployment):**
```yaml
# In your deployment's pod template metadata:
annotations:
  sensor.crowdstrike.com/imageanalyzer: "disabled"
```

---

## Upgrading

### Upgrade KAC

```bash
export KAC_IMAGE_TAG="<new_version>.container.x86_64.Release.US-1"

helm upgrade falcon-kac crowdstrike/falcon-kac \
  -n falcon-kac \
  --reuse-values \
  --set image.tag="${KAC_IMAGE_TAG}"
```

### Upgrade IAR

Update the `tag` in `iar-values.yaml`, then:

```bash
helm upgrade imageanalyzer crowdstrike/falcon-image-analyzer \
  -n falcon-image-analyzer \
  -f iar-values.yaml
```

---

## Uninstall

```bash
# Remove IAR
helm uninstall imageanalyzer -n falcon-image-analyzer
kubectl delete namespace falcon-image-analyzer

# Remove KAC
helm uninstall falcon-kac -n falcon-kac
kubectl delete namespace falcon-kac
```

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| KAC pod not starting | `kubectl describe pod -n falcon-kac` — verify CID is correct and image pull is succeeding |
| Webhook blocking all pods | Webhook `failurePolicy` is `Ignore` by default; check KAC policy in Falcon console |
| IAR pod in `Pending` | Check node resources and `priorityClassName`; verify PSS labels on namespace |
| IAR not scanning images | Confirm `deployment.enabled: true` and `daemonset.enabled: false`; check logs for auth errors |
| No scan results in console | Verify `clientID`/`clientSecret` scopes include Falcon Container Image; check outbound firewall rules to upload servers |
| KAC and IAR not communicating | Confirm IAR namespace matches `falconImageAnalyzerNamespace` in KAC values; verify IAR agent service is running on port 8001 |
| `clusterName` mismatch | Leave `clusterName` blank in IAR if KAC is running — KAC's discovered name takes precedence in the Falcon dashboard |

---

## Key Namespaces and Resources

| Resource | Namespace | Notes |
|----------|-----------|-------|
| KAC Deployment | `falcon-kac` | 1 pod, 2 containers (`falcon-client`, `falcon-ac`) |
| KAC ValidatingWebhookConfiguration | cluster-scoped | Registered automatically; excludes `kube-system`, `kube-public`, `falcon-kac`, `falcon-system` |
| IAR Deployment | `falcon-image-analyzer` | Always 1 replica in watcher mode |
| IAR Agent Service | `falcon-image-analyzer` | Port 8001 — used by KAC for scan data requests |
| IAR Temp Volume | `falcon-image-analyzer` | `emptyDir` 20Gi by default; holds extracted image layers during scanning |
