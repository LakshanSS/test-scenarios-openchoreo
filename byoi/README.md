# Deploy a Prebuilt Container Image (BYOI)

This sample demonstrates deploying prebuilt container images to OpenChoreo without using the Workflow Plane. This is useful when you have existing images built by external CI/CD pipelines.

## Overview

OpenChoreo supports deploying applications from prebuilt container images (Bring Your Own Image). You can deploy from:

- **Public registries** — No additional configuration needed
- **Private registries** — Requires storing pull credentials and a ClusterComponentType with `imagePullSecrets` support

## Prerequisites

- k3d cluster with OpenChoreo installed (single-cluster setup)
- `kubectl` configured to the cluster
- A container image to deploy

For private registry deployments, additionally:
- Docker Hub account with a private repository
- Docker Hub Personal Access Token with **Read** (or Read & Write) scope: [create one here](https://hub.docker.com/settings/personal-access-tokens)

## File Structure

```
byoi/
├── 01-public-image.yaml                 # Component + Workload for a public image
├── 02-service-private-registry-cct.yaml # ClusterComponentType with imagePullSecrets support
└── 03-private-image.yaml                # Component + Workload for a private image
```

---

## Option A — Deploy from a Public Registry

Apply the Component and Workload:

```bash
kubectl apply -f 01-public-image.yaml
```

Edit the file to replace `nginx:latest` with your own image and `80` with the port your application listens on.

### Verify

```bash
kubectl get component my-app -n default
kubectl get workload my-app -n default
kubectl get pods -A | grep my-app
```

### Test

```bash
curl http://development-default.openchoreoapis.localhost:19080/my-app-http
```

---

## Option B — Deploy from a Private Registry

### Step 1 — Store Pull Credentials in OpenBao

Generate a base64-encoded `username:token` string:

```bash
echo -n "YOUR_DOCKERHUB_USERNAME:YOUR_DOCKERHUB_TOKEN" | base64
```

Store the credentials:

```bash
kubectl exec -n openbao openbao-0 -- sh -c '
  export BAO_ADDR=http://127.0.0.1:8200 BAO_TOKEN=root
  bao kv put secret/registry-credentials \
    value="{\"auths\":{\"https://index.docker.io/v1/\":{\"auth\":\"<BASE64_AUTH>\"}}}"
'
```

Replace `<BASE64_AUTH>` with the output from the previous command.

Verify:

```bash
kubectl exec -n openbao openbao-0 -- sh -c '
  export BAO_ADDR=http://127.0.0.1:8200 BAO_TOKEN=root
  bao kv get secret/registry-credentials
'
```

---

### Step 2 — Apply the ClusterComponentType

Apply the `service-private-registry` ClusterComponentType, which adds an `imagePullSecrets` ExternalSecret so pods can pull from your private registry:

```bash
kubectl apply -f 02-service-private-registry-cct.yaml
```

> **Note:** Skip this step if you have already applied this CCT from another sample (e.g. `publish-to-private-container-registry`).

Verify:

```bash
kubectl get clustercomponenttype service-private-registry
```

---

### Step 3 — Deploy the Component and Workload

Edit `03-private-image.yaml` and replace `YOUR_DOCKERHUB_USERNAME/YOUR_PRIVATE_IMAGE:v1` with your actual private image reference. Then apply:

```bash
kubectl apply -f 03-private-image.yaml
```

### Verify

Check that the `registry-pull-secret` ExternalSecret was synced to the workload namespace:

```bash
kubectl get externalsecret -A | grep registry-pull-secret
kubectl get secret -A | grep registry-pull-secret
```

Check that pods are running:

```bash
kubectl get pods -A | grep my-private-app
```

If pods are in `ImagePullBackOff`, see the Troubleshooting section below.

### Test

```bash
curl http://development-default.openchoreoapis.localhost:19080/my-private-app-http
```

---

## Cleanup

### Public image

```bash
kubectl delete -f 01-public-image.yaml
```

### Private image

```bash
kubectl delete -f 03-private-image.yaml
kubectl delete -f 02-service-private-registry-cct.yaml
```

Remove the pull credentials from OpenBao:

```bash
kubectl exec -n openbao openbao-0 -- sh -c '
  export BAO_ADDR=http://127.0.0.1:8200 BAO_TOKEN=root
  bao kv delete secret/registry-credentials
'
```

> **Note:** Deleting the Component automatically cleans up the `registry-pull-secret` ExternalSecret and Kubernetes secret in the workload namespace.

---

## Troubleshooting

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

### Component or Workload not progressing

Check component conditions:

```bash
kubectl get component my-private-app -n default -o jsonpath='{.status.conditions}' | jq .
```

Check workload conditions:

```bash
kubectl get workload my-private-app -n default -o jsonpath='{.status.conditions}' | jq .
```
