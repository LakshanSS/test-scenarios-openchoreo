# Private Container Registry as Build Output Target

This sample demonstrates building a component from source and pushing the resulting image to a **private Docker Hub repository** as the build output target, rather than the default ephemeral `ttl.sh` registry.

## Overview

By default, OpenChoreo's workflow plane pushes built images to `ttl.sh` (an anonymous ephemeral registry). This sample shows how to replace that with Docker Hub so built images:
- Are stored persistently in your Docker Hub account
- Can be pulled from a private repository using configured credentials

**What gets configured:**
1. `publish-image` ClusterWorkflowTemplate — replaced to push to Docker Hub
2. `registry-push-secret` in OpenBao — Docker Hub push credentials for the workflow plane
3. `registry-credentials` in OpenBao — Docker Hub pull credentials for the data plane
4. `service-private-registry` ClusterComponentType — adds an `imagePullSecrets` ExternalSecret so pods can pull the private image

## Prerequisites

- k3d cluster with OpenChoreo installed (single-cluster setup)
- `kubectl` configured to the cluster
- `jq` installed (used in monitoring commands)
- Docker Hub account with a repository (can be private)
- Docker Hub Personal Access Token with **Read & Write** scope (required for pushing images): [create one here](https://hub.docker.com/settings/personal-access-tokens)
  > **Note:** Read-only tokens will cause `access token has insufficient scopes` errors during the build's publish step.

## File Structure

```
publish-to-private-container-registry/
├── 01-publish-image-dockerhub.yaml          # Replaces the publish-image CWT to push to Docker Hub
├── 02-service-private-registry-cct.yaml     # ClusterComponentType with imagePullSecrets support
├── 03-greeting-service-component.yaml       # Sample Component
└── 04-greeting-service-workflowrun.yaml     # WorkflowRun that triggers the build
```

---

## Step 1 — Prepare Credentials

Generate a base64-encoded `username:token` string for Docker Hub:

```bash
echo -n "YOUR_DOCKERHUB_USERNAME:YOUR_DOCKERHUB_TOKEN" | base64
```

Save the output — you'll use it as `<BASE64_AUTH>` in the steps below.

---

## Step 2 — Store Push Credentials in OpenBao

The workflow plane uses `registry-push-secret` to authenticate when pushing the built image to Docker Hub.

```bash
kubectl exec -n openbao openbao-0 -- sh -c '
  export BAO_ADDR=http://127.0.0.1:8200 BAO_TOKEN=root
  bao kv put secret/registry-push-secret \
    value="{\"auths\":{\"https://index.docker.io/v1/\":{\"auth\":\"<BASE64_AUTH>\"}}}"
'
```

Replace `<BASE64_AUTH>` with the base64 string from Step 1.

Verify the secret was stored:

```bash
kubectl exec -n openbao openbao-0 -- sh -c '
  export BAO_ADDR=http://127.0.0.1:8200 BAO_TOKEN=root
  bao kv get secret/registry-push-secret
'
```

---

## Step 3 — Store Pull Credentials in OpenBao

The data plane uses `registry-credentials` to pull the private image when deploying pods. You can use the same `<BASE64_AUTH>` value.

```bash
kubectl exec -n openbao openbao-0 -- sh -c '
  export BAO_ADDR=http://127.0.0.1:8200 BAO_TOKEN=root
  bao kv put secret/registry-credentials \
    value="{\"auths\":{\"https://index.docker.io/v1/\":{\"auth\":\"<BASE64_AUTH>\"}}}"
'
```

> **Note:** If your Docker Hub repository is **public**, you can skip Steps 3 and skip the `registry-pull-secret` ExternalSecret portion in `02-service-private-registry-cct.yaml`. However, the sample assumes a private repository for a complete end-to-end test.

---

## Step 4 — Configure the Publish-Image CWT

Edit `01-publish-image-dockerhub.yaml` and replace `YOUR_DOCKERHUB_USERNAME` with your actual Docker Hub username:

```yaml
REGISTRY_ENDPOINT="docker.io/YOUR_DOCKERHUB_USERNAME"
```

Then apply it. This **replaces** the cluster-wide `publish-image` CWT, so all subsequent builds will push to Docker Hub.

> **Warning:** This affects all components in the cluster. If you have other components using the default `ttl.sh` registry, they will now push to Docker Hub after this change.

```bash
kubectl apply -f 01-publish-image-dockerhub.yaml
```

Verify:

```bash
kubectl get clusterworkflowtemplate publish-image -o yaml | grep REGISTRY_ENDPOINT
```

---

## Step 5 — Apply the ClusterComponentType

Apply the custom ClusterComponentType that includes `imagePullSecrets` so the data plane can pull from your private Docker Hub repo:

```bash
kubectl apply -f 02-service-private-registry-cct.yaml
```

Verify:

```bash
kubectl get clustercomponenttype service-private-registry
```

---

## Step 6 — Deploy the Component and Trigger Build

Apply the Component first, wait for it to be registered, then apply the WorkflowRun:

```bash
kubectl apply -f 03-greeting-service-component.yaml
kubectl wait component greeting-service -n default \
  --for=jsonpath='{.metadata.uid}' --timeout=30s
kubectl apply -f 04-greeting-service-workflowrun.yaml
```

> **Why the wait?** The WorkflowRun controller validates that the referenced Component exists at reconciliation time. Applying both simultaneously can cause a `ComponentValidationFailed` race condition where the WorkflowRun is processed before the Component is registered, causing it to fail immediately without submitting an Argo workflow.

This deploys a simple Go greeter service (source: [openchoreo/sample-workloads](https://github.com/openchoreo/sample-workloads/tree/main/service-go-greeter)) and immediately triggers a build.

---

## Step 7 — Monitor the Build

Check the WorkflowRun status:

```bash
kubectl get workflowrun greeting-service-build-01 -n default -o jsonpath='{.status.conditions}' | jq .
```

Watch the workflow pods (builds run in the `workflows-default` namespace):

```bash
kubectl get pods -n workflows-default | grep greeting-service
```

View the publish-image step logs to confirm the push to Docker Hub:

```bash
# Get the pod name for the publish step
POD_NAME=$(kubectl get pods -n workflows-default -l component-workflows.argoproj.io/workflow=greeting-service-build-01 --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
kubectl logs -n workflows-default $POD_NAME -f
```

Wait until the WorkflowRun shows all three conditions as `True`:
- `WorkflowCompleted: True`
- `WorkflowSucceeded: True`
- `WorkloadUpdated: True`

---

## Step 8 — Verify the Image in Docker Hub

Once the workflow completes, check the Workload to see the image reference:

```bash
kubectl get workload -n default | grep greeting-service
kubectl get workload greeting-service -n default -o jsonpath='{.spec.container.image}'
```

The image should be in the format:
```
docker.io/YOUR_DOCKERHUB_USERNAME/greeting-service:latest-<git-revision>
```

You can also verify in Docker Hub by visiting:
`https://hub.docker.com/r/YOUR_DOCKERHUB_USERNAME/greeting-service/tags`

---

## Step 9 — Verify the Deployment

Check that the pull secret ExternalSecret was synced:

```bash
kubectl get externalsecret -A | grep registry-pull-secret
kubectl get secret -A | grep registry-pull-secret
```

Check that pods are running and successfully pulled the image from Docker Hub:

```bash
kubectl get deployment -A -l openchoreo.dev/component=greeting-service
kubectl get pods -A -l openchoreo.dev/component=greeting-service
```

Check ReleaseBinding status:

```bash
kubectl get releasebinding greeting-service-development -n default -o jsonpath='{.status.conditions}' | jq .
```

---

## Step 10 — Test the Application

```bash
curl http://development-default.openchoreoapis.localhost:19080/greeting-service-greeter-api/greeter/greet
```

Expected response:
```json
{"message": "Hello, World!"}
```

---

## Cleanup

Remove the WorkflowRun and Component. The WorkflowRun deletion automatically cleans up its scoped `registry-push-secret` ExternalSecret in `workflows-default`. The Component deletion cleans up the Workload, ReleaseBinding, and `registry-pull-secret` ExternalSecrets in the dataplane namespaces:

```bash
kubectl delete -f 04-greeting-service-workflowrun.yaml
kubectl delete -f 03-greeting-service-component.yaml
```

Remove the ClusterComponentType:

```bash
kubectl delete -f 02-service-private-registry-cct.yaml
```

Restore the default `publish-image` CWT (pointing back to ttl.sh):

```bash
kubectl apply -f https://raw.githubusercontent.com/openchoreo/openchoreo/main/samples/getting-started/workflow-templates/publish-image.yaml
```

Remove the secrets from OpenBao:

```bash
kubectl exec -n openbao openbao-0 -- sh -c '
  export BAO_ADDR=http://127.0.0.1:8200 BAO_TOKEN=root
  bao kv delete secret/registry-push-secret
  bao kv delete secret/registry-credentials
'
```

Optionally, delete the image from Docker Hub if you no longer need it:

```
https://hub.docker.com/r/YOUR_DOCKERHUB_USERNAME/greeting-service/tags
```

---

## Troubleshooting

### Workflow push fails with "unauthorized" or "insufficient scopes"

If the error is `access token has insufficient scopes`, your Docker Hub token was created with read-only access. Create a new token with **Read & Write** scope at [hub.docker.com/settings/personal-access-tokens](https://hub.docker.com/settings/personal-access-tokens) and re-run Steps 1–2 to update the secret.

If the error is a general `unauthorized`, verify the push secret is stored correctly:
```bash
kubectl exec -n openbao openbao-0 -- sh -c '
  export BAO_ADDR=http://127.0.0.1:8200 BAO_TOKEN=root
  bao kv get secret/registry-push-secret
'
```

Check that the scoped push secret Kubernetes secret was synced to the `workflows-default` namespace:
```bash
kubectl get secret -n workflows-default | grep registry-push-secret
```

### Pods in ImagePullBackOff

The pull credentials are not set up correctly. Check:
```bash
kubectl describe externalsecret registry-pull-secret -n <workload-namespace>
kubectl get events -n <workload-namespace> | grep pull
```

Confirm the `registry-credentials` secret exists in OpenBao:
```bash
kubectl exec -n openbao openbao-0 -- sh -c '
  export BAO_ADDR=http://127.0.0.1:8200 BAO_TOKEN=root
  bao kv get secret/registry-credentials
'
```

### Image pushed to wrong path

Check what image the workflow actually pushed by inspecting the Workload:
```bash
kubectl get workload greeting-service -n default -o yaml
```

The `spec.container.image` field shows the full image reference including registry and tag.

### WorkflowRun fails immediately with `ComponentValidationFailed`

If `kubectl get workflowrun greeting-service-build-01 -n default -o jsonpath='{.status.conditions}'` shows `component "greeting-service" not found`, the WorkflowRun was reconciled before the Component was registered. Delete and re-apply in the correct order:

```bash
kubectl delete workflowrun greeting-service-build-01 -n default
kubectl wait component greeting-service -n default \
  --for=jsonpath='{.metadata.uid}' --timeout=30s
kubectl apply -f 04-greeting-service-workflowrun.yaml
```

### WorkflowRun stuck or failed

Check the full workflow logs:
```bash
kubectl get pods -n workflows-default -l component-workflows.argoproj.io/workflow=greeting-service-build-01
kubectl logs -n workflows-default <pod-name> --all-containers
```
