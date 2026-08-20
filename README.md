# myapp — Secrets on EKS via AWS Secrets Manager + External Secrets Operator + ArgoCD

Kubernetes manifests and Helm chart for deploying `myapp`'s microservices to EKS, with application secrets sourced from **AWS Secrets Manager**, synced into the cluster by the **External Secrets Operator (ESO)**, and deployed via **ArgoCD**.

No plaintext secrets ever live in this repo or in Helm `values.yaml` — only secret *names* are referenced. Actual values are pulled at runtime from Secrets Manager.

---

## Architecture

```
AWS Secrets Manager (dev/myapp/all-secrets)
        │  (IAM role via IRSA)
        ▼
External Secrets Operator (in-cluster)
        │  (creates/refreshes)
        ▼
Native K8s Secrets (apigateway-secret, hotelservice-secret, ratingservice-secret,
                     registry-secret, userservice-secret)
        │  (envFrom / secretKeyRef)
        ▼
Your microservice Pods  ◄── deployed/synced by ArgoCD from this Helm chart
```

| Component | Role |
|---|---|
| **AWS Secrets Manager** | Single source of truth for secret values (`dev/myapp/all-secrets`) |
| **IRSA** | Grants ESO's ServiceAccount least-privilege IAM access to that one secret |
| **External Secrets Operator** | Watches `ExternalSecret` resources, materializes native `Secret` objects, refreshes them on an interval |
| **ArgoCD** | Continuously syncs this repo's Helm chart to the `dev` namespace |

---

## Repo structure

```
.
├── charts/myapp/
│   ├── templates/
│   │   ├── deployment.yaml
│   │   ├── external-secrets.yaml   # ExternalSecret per microservice
│   │   └── ...
│   └── values.yaml
├── cluster-secret-store.yaml
├── argocd-application.yaml
└── docs/
    └── eks-secrets-argocd-runbook.md
```

---

## Services

| Service | Secret name | AWS Secrets Manager key path |
|---|---|---|
| `apigatewayservice` | `apigateway-secret` | `apigatewayservice` |
| `hotelservice` | `hotelservice-secret` | `hotelservice` |
| `ratingservice` | `ratingservice-secret` | `ratingservice` |
| `serviceregistry` | `registry-secret` | `serviceregistry` |
| `userservice` | `userservice-secret` | `userservice` |

All five are keys within the single JSON secret `dev/myapp/all-secrets` — see the runbook for the exact structure.

---

## Prerequisites

- An existing EKS cluster (account `237974319000`, region `ap-south-1`)
- `kubectl`, `aws` CLI (admin/cluster-admin profile), `helm`, `eksctl` installed locally
- ArgoCD CLI (`argocd`) if you want to sync/inspect from the command line

---

## Quick start

Full step-by-step instructions, including all commands and manifests, are in [`docs/eks-secrets-argocd-runbook.md`](docs/eks-secrets-argocd-runbook.md). Summary:

1. **Create the secret** in AWS Secrets Manager (`dev/myapp/all-secrets`).
2. **Create an IAM policy** scoped to read only that secret.
3. **Confirm the OIDC provider** is associated with the cluster (`eksctl utils associate-iam-oidc-provider`).
4. **Create the IRSA ServiceAccount** (`eksctl create iamserviceaccount`) for ESO.
5. **Install ESO** via Helm, pointed at the IRSA ServiceAccount.
6. **Apply the `ClusterSecretStore`** (`cluster-secret-store.yaml`).
7. **Deploy `ExternalSecret` resources** (one per microservice, templated in `charts/myapp/templates/external-secrets.yaml`).
8. **Deploy ArgoCD** and create the `Application` (`argocd-application.yaml`) pointing at `charts/myapp`.
9. **Verify** pods, `ExternalSecret` sync status, and populated env vars.

```bash
kubectl apply -f cluster-secret-store.yaml
kubectl apply -f argocd-application.yaml
argocd app sync myapp-dev
kubectl get pods -n dev
```

---

## Rotating secrets

Update the value in Secrets Manager — no redeploy required:

```bash
aws secretsmanager put-secret-value \
  --secret-id dev/myapp/all-secrets \
  --region ap-south-1 \
  --secret-string file://all-secrets.json
```

ESO refreshes the K8s `Secret` within `refreshInterval` (configurable per environment via `externalSecrets.refreshInterval` in `values.yaml`). Since pods don't hot-reload env vars, add [Reloader](https://github.com/stakater/Reloader) if you want automatic rolling restarts on secret change.

---

## Troubleshooting

If a pod's env var is missing or empty:

1. `kubectl get externalsecret <name> -n dev -o yaml` → check `status.conditions`
2. `kubectl logs -n external-secrets deploy/external-secrets` → IAM/auth errors
3. `kubectl get secret <name> -n dev -o yaml` → confirm keys exist
4. Confirm `envFrom.secretRef.name` in the Deployment matches `target.name` in the `ExternalSecret`

---

## License

Internal project — add a license here if this repo will be public.
