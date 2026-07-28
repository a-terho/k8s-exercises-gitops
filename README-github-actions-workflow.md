## Create GitHub Actions workflow job

Follow the initial instructions in the [README.md](./README.md) file.

Create a GitHub actions workflow job that runs after build and publish job. Add the job for both staging and production release workflows, pointing to the correct `kustomization.yaml` file in GitOps repository. Replace project specific placeholder `PROJECT_ID` and Artifact Registry specific placeholders (`REGISTRY`, `REPOSITORY`) accordingly.

```yaml
env:
  RELEASE_TAG: 'main-${{ github.sha }}' # or any relevan name

jobs:
  build-and-publish: ...

  update-kustomization:
    name: Update kustomization.yaml
    runs-on: ubuntu-latest
    needs: build-publish

    # Load required environment variables from previous job using outputs
    # or define names here. Assuming GKE Artifact Registry is used, container
    # image name format is REGISTRY/PROJECT_ID/REPOSITORY/<application name>
    env:
      BROADCASTER_IMAGE: '${{ needs.build-publish.outputs.BROADCASTER_IMAGE }}'
      TODO_APP_IMAGE: '${{ needs.build-publish.outputs.TODO_APP_IMAGE }}'
      TODO_BACKEND_IMAGE: '${{ needs.build-publish.outputs.TODO_BACKEND_IMAGE }}'

    steps:
      - name: Checkout
        uses: actions/checkout@v6
        with:
          repository: a-terho/k8s-exercises-gitops
          token: ${{ secrets.GITOPS_TOKEN }}
          ref: main # update file on main branch

      - name: Set up Kustomize
        uses: imranismail/setup-kustomize@v3

      - name: Modify kustomization.yaml to use right images
        working-directory: overlays/staging
        run: |-
          kustomize edit set image "$BROADCASTER_IMAGE:$RELEASE_TAG"
          kustomize edit set image "$TODO_APP_IMAGE:$RELEASE_TAG"
          kustomize edit set image "$TODO_BACKEND_IMAGE:$RELEASE_TAG"

      - name: Commit kustomization.yaml to GitHub
        uses: EndBug/add-and-commit@v10
        with:
          cwd: './overlays/staging'
          add: 'kustomization.yaml'
          message: New staging version of project released
          push: origin main # push to the main branch
```

Generate the `GITOPS_TOKEN` from Profile **Settings** -> **Developer settings** -> **Personal access tokens** -> **Fine-grained tokens** -> **Generate new token**. Give token a name, expiration time, choose **Only select repositories** and select the GitOps repository and give **Repository permissions** to **Contents** with **Read and write**. Then add the PAT to Secrets in the Git repository that runs the workflow.

Now when a new container image gets published, the corresponding image name is updated by the GitHub Actions workflow to `kustomization.yaml` file in the GitOps repository.
