<!-- GitOps repository holding plain Kubernetes manifests for ShopKart on Amazon EKS. -->
<!-- Synchronized automatically by Argo CD and updated by the Jenkins CI pipeline from shopkart-app. -->

# ShopKart GitOps

Declarative plain Kubernetes manifests (no Helm, no Kustomize) for running the **ShopKart** application stack on Amazon EKS, continuously deployed and managed by Argo CD.

---

## 🗺️ ShopKart Project Map

| Repository | Role & Description | Link | Status |
| :--- | :--- | :--- | :--- |
| **`shopkart-app`** | Application source code (React SPA frontend, Node.js REST API backend), Dockerfiles, and Jenkins CI pipeline. | [github.com/vkashsingh/shopkart-app](https://github.com/vkashsingh/shopkart-app) | Upstream App |
| **`shopkart-infra`** | Infrastructure as Code (Terraform) provisioning AWS VPC, EKS Cluster, IAM roles, and base cluster add-ons. | [github.com/vkashsingh/shopkart-infra](https://github.com/vkashsingh/shopkart-infra) | Infrastructure |
| **`shopkart-gitops`** | GitOps repository containing plain Kubernetes manifests and Argo CD Application definitions. | [github.com/vkashsingh/shopkart-gitops](https://github.com/vkashsingh/shopkart-gitops) | **You are here: shopkart-gitops** |

---

## 🔄 Deployment Flow

Deployments to Amazon EKS follow an automated GitOps delivery workflow:

```
[ Developer ]
      │ (git push)
      ▼
[ shopkart-app ]
      │ (triggers build)
      ▼
[ Jenkins CI ]
      │ 1. Run unit & integration tests
      │ 2. Build & push container images (vkashs/shopkart-*:SHA)
      │ 3. Update image tags in shopkart-gitops manifests
      ▼ (git commit & push)
[ shopkart-gitops ] ◀── Single Source of Truth
      │
      │ (continuous reconciliation / webhook)
      ▼
[ Argo CD (EKS) ]
      │ Automated sync (prune=true, selfHeal=true)
      ▼
[ Amazon EKS Cluster ] (namespace: shopkart)
      ├── StatefulSet: mongo (gp3 EBS PVC)
      ├── Deployment: shop-backend (2 replicas, PDB)
      ├── Deployment: shop-frontend (2 replicas)
      └── Service: shop-frontend (LoadBalancer)
```

1. **Continuous Integration (`shopkart-app`)**: Code commits trigger the Jenkins pipeline, which executes tests and builds container images (`vkashs/shopkart-backend:<commit-sha>` and `vkashs/shopkart-frontend:<commit-sha>`).
2. **GitOps Update (`shopkart-gitops`)**: Jenkins automatically updates the image tag lines inside `manifests/backend-deployment.yaml` and `manifests/frontend-deployment.yaml`, then commits and pushes changes back to the `main` branch.
3. **Continuous Deployment (`Argo CD`)**: Argo CD detects the updated manifests, reconciles the EKS cluster to match the Git state, automatically creates/updates workloads, prunes orphaned resources, and continuously self-heals cluster drift.

---

## 🔐 One-Time Secret Setup

Workload manifests in this repository do not store credentials. The `shopkart-mongo` Secret must be created once in the cluster before pods initialize.

Run the following one-time `kubectl` command using placeholders (replace `<USER>` and `<PASS>` with your database credentials):

```bash
kubectl create secret generic shopkart-mongo \
  --namespace=shopkart \
  --from-literal=mongo-root-username='<USER>' \
  --from-literal=mongo-root-password='<PASS>' \
  --from-literal=mongo-url='mongodb://<USER>:<PASS>@mongo:27017/shopkart?authSource=admin'
```

- `mongo-statefulset.yaml` references `mongo-root-username` and `mongo-root-password` via `secretKeyRef` to set initial root credentials.
- `backend-deployment.yaml` references `mongo-url` via `secretKeyRef` to supply the MongoDB connection string to the backend service.

---

## ⏪ Rollback Procedure

Because Git is the sole source of truth for the cluster, rollbacks are performed via Git history rather than ad-hoc cluster changes:

1. **Locate the Target Commit**:
   Find the last known good commit SHA:
   ```bash
   git log --oneline
   ```

2. **Revert the Commit**:
   Revert the faulty commit to generate an inverse commit:
   ```bash
   git revert <COMMIT_SHA>
   ```

3. **Push the Revert**:
   Push the revert commit to the `main` branch:
   ```bash
   git push origin main
   ```

4. **Argo CD Reconciles**:
   Argo CD will automatically detect the revert on `main` and sync the cluster back to the previous stable state. To trigger immediate synchronization:
   ```bash
   argocd app sync shopkart
   ```

---

## 📂 Repository File Tree

```
shopkart-gitops/
├── README.md
├── apps/
│   └── shopkart-app.yaml
└── manifests/
    ├── 00-namespace.yaml
    ├── backend-deployment.yaml
    ├── backend-pdb.yaml
    ├── backend-service.yaml
    ├── frontend-deployment.yaml
    ├── frontend-service.yaml
    ├── mongo-service.yaml
    └── mongo-statefulset.yaml
```

