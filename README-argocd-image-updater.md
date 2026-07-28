## Install ArgoCD ImageUpdater

Follow the initial instructions in the [README.md](./README.md) file.

Because ImageUpdater works by polling the container registry, you need to configure [registries-patch.yaml](argocd-image-updater/registries-patch.yaml) to have it point to correct `REGISTRY_LOCATION-docker.pkg.dev` for fields `prefix` and `api_url`.

[registries-patch.yaml](argocd-image-updater/registries-patch.yaml) patches ArgoCD with custom registry configuration ([ref](https://argocd-image-updater.readthedocs.io/en/stable/configuration/registries/#configuring-custom-registries)) to use a shell script ([gcr-auth-script.yaml](argocd-image-updater/gcr-auth-script.yaml)) to periodically generate credentials (OAuth access tokens) ([ref](https://argocd-image-updater.readthedocs.io/en/stable/basics/authentication/#using-a-script-to-generate-credentials)) using an approach which fetches them from GKE cluster's own metadata server with `wget` (`curl` is not available) ([ref](https://docs.cloud.google.com/compute/docs/access/authenticate-workloads#applications)). Using `docker login` with OAuth access tokens authenticates to user `oauth2accesstoken`. This shell script is then patched onto `argocd-image-updater-controller` deployment.

Install ArgoCD ImageUpdater with:

```bash
kubectl apply -k argocd-image-updater
```

## Authenticate ArgoCD ImageUpdater

ImageUpdater needs a way to push changes to the Git repository. Generate SSH keypair with the following command and create `secret.yaml` file containing the private generated inside `deploy-key` file.

```bash
ssh-keygen -t ed25519 -a 100 -C "argocd-image-updater-deploy-key" -f ./deploy-key -N ""
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-credentials-secret
  namespace: argocd
type: Opaque
stringData:
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

Add the contents of `deploy-key.sub` to Git repository **Settings** -> **Deploy keys** -> **Add deploy key** and check **Allow write access**. Then, deploy the Secret to the cluster with following command:

```bash
kubectl apply -f secret.yaml
```

In order for ImageUpdater to be able to pull images from Google Cloud Artifact Registry, you need to authorize it with a service account. This setup uses service account impersonation through GKE Workload Identity Federation. Replace cluster and project specific placeholders (`CLUSTER_NAME`, `CLUSTER_LOCATION`, `PROJECT_ID`) and Artifact Registry specific placeholders (`REPOSITORY_NAME`, `REGISTRY_LOCATION`) accordingly.

```bash
# Enable Workload Identity Federation for the GKE cluster (if not enabled already)
gcloud container clusters update CLUSTER_NAME --location=CLUSTER_LOCATION --workload-pool=PROJECT_ID.svc.id.goog

# Create Google Cloud IAM service account for ArgoCD ImageUpdater
gcloud iam service-accounts create "argocd-image-updater-sa" \
  --display-name="ArgoCD ImageUpdater Service Account"

# Give IAM service account read access to Artifact Registry
gcloud artifacts repositories add-iam-policy-binding REPOSITORY_NAME --location=REGISTRY_LOCATION \
  --member="serviceAccount:argocd-image-updater-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"
```

ArgoCD ImageUpdater installation creates a Kubernetes service account called `argocd-image-updater-controller` ([ref](https://argocd-image-updater.readthedocs.io/en/stable/basics/authentication/#authentication-to-kubernetes)). It can be annotated to map requests to (and impersonate as) Google IAM service account with following commands.

```bash
kubectl annotate serviceaccount -n argocd argocd-image-updater-controller \
  iam.gke.io/gcp-service-account=argocd-image-updater-sa@PROJECT_ID.iam.gserviceaccount.com

gcloud iam service-accounts add-iam-policy-binding \
  argocd-image-updater-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:PROJECT_ID.svc.id.goog[argocd/argocd-image-updater-controller]"
```

After installation ArgoCD ImageUpdater continues polling the container registry until the installation is removed.
