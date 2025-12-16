# Argo Events ➜ Workflows ➜ Argo CD ➜ Rollouts Lab

This repository contains everything you need to practice wiring a full Argo stack: a GitHub push is forwarded through Smee.io to Argo Events, a sensor starts an Argo Workflow that builds and publishes an image, the workflow updates the Helm chart, and Argo CD reconciles the change by deploying an Argo Rollout onto Minikube.

```
GitHub push → Smee webhook relay → Argo EventSource → Sensor → Argo Workflow
        → GHCR image + Git commit → Argo CD ApplicationSet → Argo Rollout on Minikube
```

## Learning goals

- Configure Argo Events to receive GitHub pushes via Smee.io.
- Author a Workflow that builds an image (Kaniko), pushes it to GitHub Container Registry (GHCR), and commits a Helm values update.
- Use Argo CD + ApplicationSet to continuously deliver the chart as an Argo Rollout to multiple namespaces.
- Observe progressive delivery behavior with the Argo Rollouts controller.

## Repository map

| Path | Description |
| --- | --- |
| `apps/my-service/Dockerfile` | Minimal NGINX-based image that serves `app/index.html`. |
| `apps/my-service/helm/` | Helm chart that renders an `argoproj.io/v1alpha1` Rollout plus Service. |
| `argocd/` | Root Application and ApplicationSet manifests for Argo CD. |
| `argo-events/event-source.yaml` | GitHub webhook EventSource with an embedded Smee relay sidecar. |
| `argo-events/sensor.yaml` | Sensor that triggers the workflow template from push events. |
| `apps/smee-relay/Dockerfile` | Build context for the lightweight Smee relay container image. |

The Helm chart already ships with a canary Rollout (`apps/my-service/helm/templates/rollout.yaml`). Argo CD points at `apps/my-service/helm` and uses the correct environment specific `values-*.yaml` files.

---

## Prerequisites

- macOS/Linux shell with `kubectl`, `helm`, `minikube`, `docker` (or another container runtime), and `jq`.
- `argo` CLI (`brew install argocd argo-rollouts` or see docs).
- GitHub personal access token (PAT) with `repo`, `workflow`, and `write:packages` scopes.
- GitHub Container Registry (GHCR) repository such as `ghcr.io/<your-user>/lab24-rollouts`.
- Permission to build and push the Smee relay image defined in `apps/smee-relay/Dockerfile` (publishing to GHCR keeps everything in one registry).
- Fork of this repo so the workflow can push commits.

> All commands assume the repo is cloned to `~/github/lab24-helm-deployment`, Minikube is the current kube-context, and you have network access to download controller manifests. Adjust paths and versions to your environment.

---

## 1. Boot Minikube and install the Argo controllers

```bash
minikube start --kubernetes-version=v1.29.0 --cpus=4 --memory=8192

# Namespaces
kubectl create namespace argocd
kubectl create namespace argo-events
kubectl create namespace argo-rollouts
kubectl create namespace argo
kubectl create namespace cicd        # workflows live here
```

Install the controllers (pin the versions appropriate for your cluster):

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -n cicd -f https://github.com/argoproj/argo-workflows/releases/latest/download/install.yaml
kubectl apply -n argo-events -f https://github.com/argoproj/argo-events/releases/latest/download/install.yaml
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

> Tip: install the `kubectl argo rollouts` plugin so you can inspect rollout status easily.

---

## 2. Prepare service accounts, roles, and secrets

### 2.1 Service account for workflows

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: workflow-runner
  namespace: cicd
imagePullSecrets:
  - name: ghcr-creds
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: workflow-role
  namespace: cicd
rules:
  - apiGroups: [""]
    resources: [pods, pods/log, pods/exec, persistentvolumeclaims, configmaps, secrets]
    verbs: [create, update, patch, delete, get, list, watch]
  - apiGroups: ["argoproj.io"]
    resources: [workflows, workflowtemplates, clusterworkflowtemplates]
    verbs: [create, update, patch, delete, get, list, watch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: workflow-rolebinding
  namespace: cicd
subjects:
  - kind: ServiceAccount
    name: workflow-runner
    namespace: cicd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: workflow-role
EOF
```

### 2.2 Registry and Git secrets

```bash
# Docker config for GHCR (replace <>)
kubectl create secret docker-registry ghcr-creds \
  --namespace cicd \
  --docker-server=ghcr.io \
  --docker-username=<github-user> \
  --docker-password=<github-personal-access-token>

# Git identity for committing values.yaml changes
kubectl create secret generic github-token \
  --namespace cicd \
  --from-literal=username=<github-user> \
  --from-literal=email=<user@example.com> \
  --from-literal=token=<github-personal-access-token>

# Shared secret used by GitHub webhook + Argo Events
kubectl create secret generic github-webhook-secret \
  --namespace argo-events \
  --from-literal=token=<long-random-string>
```

---

## 3. Author the WorkflowTemplate

The template below uses Kaniko to build/push the container image straight from the Git repo and then patches `apps/my-service/helm/values.yaml` to reflect the new image tag before committing back to `main`.

Save as `argo/workflow-template.yaml` and apply it (`kubectl apply -f argo/workflow-template.yaml`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: git-build-push
  namespace: cicd
spec:
  serviceAccountName: workflow-runner
  arguments:
    parameters:
      - name: git-repo
      - name: git-revision
      - name: image-name     # e.g. ghcr.io/<user>/lab24-rollouts
      - name: image-tag
  entrypoint: main
  templates:
    - name: main
      steps:
        - - name: build-image
            template: build-image
        - - name: update-values
            template: update-values
    - name: build-image
      inputs:
        parameters:
          - name: git-repo
          - name: git-revision
          - name: image-name
          - name: image-tag
      container:
        image: gcr.io/kaniko-project/executor:v1.16.0
        args:
          - "--context={{inputs.parameters.git-repo}}#{{inputs.parameters.git-revision}}"
          - "--destination={{inputs.parameters.image-name}}:{{inputs.parameters.image-tag}}"
          - "--dockerfile=apps/my-service/Dockerfile"
          - "--snapshotMode=redo"
          - "--cleanup"
        volumeMounts:
          - name: docker-config
            mountPath: /kaniko/.docker
      volumes:
        - name: docker-config
          secret:
            secretName: ghcr-creds
            items:
              - key: .dockerconfigjson
                path: config.json
    - name: update-values
      inputs:
        parameters:
          - name: git-repo
          - name: git-revision
          - name: image-name
          - name: image-tag
      script:
        image: alpine:3.19
        command: [sh]
        source: |
          set -euo pipefail
          apk add --no-cache git yq
          git clone "{{inputs.parameters.git-repo}}" repo
          cd repo
          git checkout "{{inputs.parameters.git-revision}}"
          yq -i ".image.repository = \"{{inputs.parameters.image-name}}\"" apps/my-service/helm/values.yaml
          yq -i ".image.tag = \"{{inputs.parameters.image-tag}}\"" apps/my-service/helm/values.yaml
          git config user.name "${GIT_USERNAME}"
          git config user.email "${GIT_EMAIL}"
          git commit -am "chore: update rollout image to {{inputs.parameters.image-tag}}"
          git remote set-url origin "https://${GIT_USERNAME}:${GIT_TOKEN}@${GIT_REMOTE}"
          git push origin HEAD:main
        env:
          - name: GIT_USERNAME
            valueFrom:
              secretKeyRef:
                name: github-token
                key: username
          - name: GIT_EMAIL
            valueFrom:
              secretKeyRef:
                name: github-token
                key: email
          - name: GIT_TOKEN
            valueFrom:
              secretKeyRef:
                name: github-token
                key: token
          - name: GIT_REMOTE
            value: github.com/<your-user>/lab24-helm-deployment.git
```

> The template assumes you always push to `main`. Use Workflow parameters if you need branch-specific behavior.

---

## 4. Configure Argo Events + in-cluster Smee relay

The repo now includes `argo-events/event-source.yaml` and `argo-events/sensor.yaml`. The EventSource uses `podSpecPatch` to run a Smee relay sidecar (built from `apps/smee-relay/Dockerfile`) that forwards GitHub webhooks directly to the webhook service—no manual port-forwarding required.

1. Browse to [https://smee.io](https://smee.io) and click **Start a new channel**. Copy the unique HTTPS URL (e.g. `https://smee.io/abc123`).
2. In your GitHub fork open **Settings → Webhooks** and add a webhook:
   - Payload URL: `https://smee.io/abc123`
   - Content type: `application/json`
   - Secret: the same value stored in `github-webhook-secret`.
   - Select **Just the push event**.
   - After saving the webhook, store the relay URL in the cluster so the sidecar can read it:
     ```bash
     kubectl create secret generic smee-relay-url \
       --namespace argo-events \
       --from-literal=url=https://smee.io/abc123
     ```
3. Build and push the relay image (replace `ghcr.io/<your-user>/smee-relay:latest` with wherever you publish images):
   ```bash
   docker build -t ghcr.io/<your-user>/smee-relay:latest apps/smee-relay
   docker push ghcr.io/<your-user>/smee-relay:latest
   ```
4. Edit `argo-events/event-source.yaml` so the `smee-relay` container image matches the reference you just pushed (and adjust the secret name if needed). Update `argo-events/sensor.yaml` with your GHCR repo name for the workflow trigger arguments.
5. Apply the manifests:
   ```bash
   kubectl apply -f argo-events/event-source.yaml
   kubectl apply -f argo-events/sensor.yaml
   ```

> The Smee sidecar reads the channel URL from the `smee-relay-url` secret and proxies to `http://127.0.0.1:12000/payload`, which is the webhook listener running in the same pod. If you change the EventSource port or endpoint, update the `SMEE_TARGET` env var in `apps/smee-relay/Dockerfile` accordingly. Update `serviceAccountName` in the manifests if you already have dedicated accounts inside `argo-events`, and tweak the Sprig `substr` call in the sensor if you prefer full commit SHAs.

---

## 5. Bootstrap Argo CD and the ApplicationSet

Update `argocd/root-app-appsets.yaml` and `argocd/applicationsets/my-service-applicationset.yaml` so `repoURL` points to **your fork** (and adjust `targetRevision` if needed).

```bash
kubectl apply -f argocd/root-app-appsets.yaml
```

Argo CD now renders the ApplicationSet, which creates one Application per environment (`dev`, `staging`, `prod`) and deploys the Helm chart. Because the chart renders an Argo Rollout, you can watch the progressive update when a new image tag is committed.

---

## 6. Run the lab

1. Commit and push a change to your fork (anything—`README` tweaks work) so GitHub fires a push event.
2. Confirm the webhook is delivered by watching the EventSource pod logs (both the main container and the `smee-relay` sidecar show activity). The sensor should immediately submit a workflow:
   ```bash
   kubectl -n argo-events logs deploy/github-webhook-eventsource -c smee-relay | tail
   argo -n cicd list
   argo -n cicd get <workflow-name>
   ```
3. When the workflow finishes, Argo CD detects the changed `values.yaml` (new `image.tag`) and performs an automated sync.
4. Watch the rollout:
   ```bash
   kubectl argo rollouts get rollout my-service-dev -n my-service-dev --watch
   ```
5. Port-forward to the service to see the HTML page served by the updated image.

If anything fails, inspect the workflow pod logs (`argo -n cicd logs -w <workflow-name> -s build-image`), and double-check that your secrets and template parameters match your fork and GHCR repository.

---

## 7. Cleanup

```bash
kubectl delete -f argo-events/sensor.yaml
kubectl delete -f argo-events/event-source.yaml
kubectl delete secret smee-relay-url -n argo-events
kubectl delete -f argocd/root-app-appsets.yaml
kubectl delete workflowtemplates.argoproj.io/git-build-push -n cicd
minikube delete
```

---

## Troubleshooting tips

- **Workflow cannot push to GHCR**: re-create `ghcr-creds` secret; confirm PAT has `write:packages`. Use `kubectl get secret ghcr-creds -n cicd -o yaml` to verify base64 data exists.
- **Workflow fails to push to GitHub**: ensure `github-token` secret contains `username`, `email`, and `token` keys. Token must allow `repo` scope.
- **Sensor does not trigger**: check `kubectl -n argo-events get eventsources,sensors,pods`. Describe the sensor to see last event.
- **Rollout stuck**: use `kubectl argo rollouts analyze ...` or check pods in the target namespace. Remember Argo Rollouts requires the CRD installed cluster-wide.

Enjoy experimenting with the full Argo toolchain!
