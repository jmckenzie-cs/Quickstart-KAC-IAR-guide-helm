# Quickstart: CrowdStrike Falcon KAC + IAR via Helm

Quick start deployment guide for deploying CrowdStrike Falcon's Kubernetes Admission Controller (KAC) and Image Assessment at Runtime (IAR) using Helm.

## Contents

| File | Description |
|------|-------------|
| [`falcon-kac-iar-quickstart.md`](./falcon-kac-iar-quickstart.md) | Step-by-step deployment guide |

## What's Covered

- **Falcon KAC** — Validates workloads at admission time, detects Indicators of Misconfiguration (IOMs), and can block non-compliant pods from scheduling
- **Falcon IAR (Watcher Mode)** — Scans running container images for vulnerabilities via the Kubernetes API without requiring node-level runtime socket access

## Who This Is For

Platform/security engineers deploying CrowdStrike Falcon to Kubernetes clusters on EKS, AKS, GKE, or K3s.

## Quick Reference

```bash
# Add the CrowdStrike Helm repo
helm repo add crowdstrike https://crowdstrike.github.io/falcon-helm
helm repo update

# Deploy KAC
helm upgrade --install falcon-kac crowdstrike/falcon-kac \
  -n falcon-kac --create-namespace \
  --set falcon.cid=<YOUR_CID> \
  --set image.repository=<KAC_IMAGE_REPO> \
  --set image.tag=<KAC_IMAGE_TAG>

# Deploy IAR (Watcher Mode)
helm upgrade --install imageanalyzer crowdstrike/falcon-image-analyzer \
  -n falcon-image-analyzer --create-namespace \
  -f iar-values.yaml
```

See [`falcon-kac-iar-quickstart.md`](./falcon-kac-iar-quickstart.md) for full prerequisites, values files, verification steps, and troubleshooting.

## Requirements

- Helm 3.x
- Kubernetes 1.25+ (EKS, AKS, GKE, K3s)
- CrowdStrike CID with checksum
- Falcon API OAuth2 credentials (Falcon Container Image scope)

## Resources

- [CrowdStrike Falcon Helm Charts](https://github.com/CrowdStrike/falcon-helm)
- [Falcon Container Sensor Pull Script](https://github.com/CrowdStrike/falcon-scripts)
- [Falcon Kubernetes Admission Controller — Official Docs](https://docs.crowdstrike.com/access?ft:originId=db127604)
- [Falcon Image Assessment at Runtime — Official Docs](https://docs.crowdstrike.com/access?ft:originId=tcb77d50)
