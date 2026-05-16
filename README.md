# tgt-manifests

GitOps repo for the tgt-eks-01 EKS Auto Mode cluster. ArgoCD watches `argocd-apps/applicationset.yaml` and auto-discovers per-app folders under `manifests/dev/`.

GitHub-side equivalent of `gitlab-terraform-gcp/tgt-gitops/tgt-02-manifests/`.

---

## Layout

```
tgt-manifests/
├── argocd-apps/
│   └── applicationset.yaml      ← AppSet — directory generator over manifests/dev/*
└── manifests/dev/
    ├── online-shop/
    │   ├── base/
    │   │   ├── namespace.yaml
    │   │   ├── deployment.yaml      ← image placeholder = bare app name
    │   │   ├── service.yaml         ← ClusterIP 80 → 8080
    │   │   ├── ingress.yaml         ← path /shop/, ALB internal, group tgt-shared
    │   │   └── kustomization.yaml
    │   └── overlays/dev/
    │       └── kustomization.yaml   ← images: newName=ECR/tgt-images, newTag=online-shop-<sha>
    └── art-gallery/                 ← same shape, /art-gallery/ path
```

## Conventions

- **Kustomize standard**: every app uses `base/` + `overlays/dev/`. Image identity lives in the overlay only (HARD RULE).
- **Single ECR repo** (HARD RULE `feedback_single_ecr_repo`): `tgt-images` houses everything; per-app discrimination via tag prefix `<app>-<sha>`.
- **Distroless UID 65532**: explicit `runAsUser: 65532` on every container (HARD RULE).
- **Shared ALB** via `alb.ingress.kubernetes.io/group.name: tgt-shared` — both apps land on ONE internal ALB with path-based routing.
- **No HealthCheckPolicy CRD** (GKE-only); ALB health-check is configured via Ingress annotation `alb.ingress.kubernetes.io/healthcheck-path`.

## How a new app onboards

1. Create `manifests/dev/<app>/base/{namespace,deployment,service,ingress,kustomization}.yaml`
2. Create `manifests/dev/<app>/overlays/dev/kustomization.yaml` with `images:` newName=ECR/tgt-images, newTag=`<app>-bootstrap`
3. Push to main on this repo
4. Within ~30s ArgoCD's AppSet generates Application `<app>` targeting namespace `<app>`
5. First sync → pods ImagePullBackOff (no `<app>-bootstrap` tag exists in ECR — expected)
6. Push code in `tgt-<app>` repo → CI pushes image to ECR as `tgt-images:<app>-<sha>`
7. **Manual bump** of overlay newTag to the new SHA (Image Updater not yet wired with ECR auth — see `tgt-cluster-iac/argocd-image-updater.tf` header)
8. ArgoCD picks up the gitops commit, pods roll

## Image Updater status

Currently a no-op for ECR (lacks aws-cli in pod). Bumps are MANUAL via:

```sh
cd manifests/dev/<app>/overlays/dev
kustomize edit set image <app>=784916389752.dkr.ecr.us-east-1.amazonaws.com/tgt-images:<app>-<SHA>
git commit -am "bump <app> to <SHA>"
git push
```

When IU ECR auth is wired (phase 2.5), bumps become automatic — the AppSet annotations are already in place.
