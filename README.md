# GitOps setup for project with ArgoCD

This setup adds both staging and production deployments with their own namespaces to the Google Kubernetes Engine (GKE) cluster. Pushes to main branch of the [k8s-exercises repository](https://github.com/a-terho/k8s-exercises) are deployed to staging environment. Tagged releases are published to production environment.

## Prepare ArgoCD

First make sure the Kubernetes cluster is running on GKE and `kubectl` points to cluster context.

Then, install ArgoCD to the cluster and choose whether to use ArgoCD ImageUpdater ot GitHub Actions workflow to update image tags in relevant `kustomization.yaml` files so that ArgoCD is able to sync deployment state correctly.

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

- Option A: ArgoCD ImageUpdater uses polling to scan for new images pushed to the container registry and updates `kustomization.yaml` files automatically in the GitOps repository to reflect matching new releases. [Follow the instructions here](./README-argocd-image-updater.md).
- Option B: GitHub Actions workflow pushes updated `kustomization.yaml` files directly from the workflow to the GitOps repository after each new release. No polling is required as the workflow triggers the update. [Follow the instructions here](./README-github-actions-workflow.md).

## Add applications

Add applications to the cluster with the following commands:

```bash
kubectl apply -f overlays/staging/application.yaml
kubectl apply -f overlays/production/application.yaml
```

The only additional resource that needs to be applied manually is the Secret manifest for the broadcaster resource. Follow the instructions in [brodcaster README.md](https://github.com/a-terho/k8s-exercises/blob/4.10/broadcaster/README.md) to create and apply `secret.yml` file directly to the relevant namespace (`staging` or `production`) with `kubectl apply -f secret.yml --namespace=<namespace>`. Using Google Cloud Key Management Service (KMS) is not required if unencrypted Secret is pushed directly to the cluster.

Database backup job runs only in `production` namespace. To prepare Google Cloud Storage permissions, follow [Authenticating database backup system guide in workflows README.md file](https://github.com/a-terho/k8s-exercises/blob/4.10/.github/workflows/README.md#authenticating-database-backup-system) but replace namespace `project` with `production`.

## Access ArgoCD

To access ArgoCD from outside the cluster (from the web interface or CLI) you can expose a service IP through LoadBalancer resouce. IP can be printed with the following command when it is available.

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

```bash
echo "http://$(kubectl get svc --namespace=argocd | grep "LoadBalancer" | awk '{print $4}')"
```

Then login to ArgoCD. Default `admin` account password can be printed with this command:

```bash
kubectl get -n argocd secrets argocd-initial-admin-secret -o yaml | grep "password:" | awk '{print $2}' | base64 -d
```
