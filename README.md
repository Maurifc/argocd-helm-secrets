# README

## Before you begin
Load env variables
```bash
PROJECT_ID=$(gcloud config get-value project)
```

> Use a cluster with Workload Identity enabled

Allow ArgoCD service account to use Workload Identity

> Check argocd namespace and service account name from Kubernetes side
```bash
gcloud iam service-accounts add-iam-policy-binding argocd@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[argocd/argocd]"
```
## Install ArgoCD
```bash
helm repo add argo https://argoproj.github.io/argo-helm
```

```bash
helm upgrade --install \
  argocd \
  argo/argo-cd \
  -n argocd \
  -f argocd/values.yaml
```

## Create KR and Key
```bash
gcloud kms keyrings create sops-keyring \
    --location global
```

```bash
gcloud kms keys create sops \
    --keyring sops-keyring \
    --location global \
    --purpose "encryption"
```

# Install Helm secrets plugin
```bash
helm plugin install https://github.com/jkroepke/helm-secrets --version v4.2.2
```

## Encode the 'values-secrets.dec.yaml'
```bash
KMS_KEY="projects/$PROJECT_ID/locations/global/keyRings/sops-keyring/cryptoKeys/sops"
```

```bash
sops --encrypt --gcp-kms  values-secrets.dec.yaml > values-secrets.enc.yaml
```

## Render the chart using the encrypted values file
```bash
helm template sample . -f values.yaml -f secrets://values-secrets.enc.yaml  | grep config -b5
```