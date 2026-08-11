---
type: Usage
title: hetzner-object-storage-kog — usage
description: How to install and use the KOG — install oasgen-provider, install this chart, then provision a bucket with the credentials Secret, a BucketConfiguration, and a Bucket CR.
resource: oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog
tags: [krateo, oasgen, install, bucket, hetzner]
timestamp: 2026-08-11T00:00:00Z
---

# Usage

## Prerequisites

- A Kubernetes cluster (1.20+).
- [`oasgen-provider`](https://github.com/krateo-platformops/oasgen-provider) installed
  in the cluster — this KOG's `RestDefinition` is inert without it:

  ```console
  $ helm repo add krateo https://charts.krateo.io
  $ helm repo update
  $ helm install oasgen-provider krateo/oasgen-provider \
      --namespace krateo-system --create-namespace
  ```

- Hetzner S3 credentials (access key + secret key). These are currently only
  generated from the Hetzner Console — there is no public API for them.

## Install the KOG

From the published OCI artifact (recommended):

```console
$ helm install hetzner-object-storage-kog \
    oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog \
    --version 0.1.0 \
    --namespace krateo-system --create-namespace
```

Or from a local checkout:

```console
$ helm install hetzner-object-storage-kog ./chart --namespace krateo-system
```

The install renders a `ConfigMap` (the OAS) and a `RestDefinition`
([overview](./overview.md)). `oasgen-provider` then reads the `RestDefinition` and
generates the CRDs:

```console
$ kubectl get restdefinition -n krateo-system
$ kubectl get crd | grep objectstorage.hetzner.ogen.krateo.io
bucket.objectstorage.hetzner.ogen.krateo.io
bucketconfiguration.objectstorage.hetzner.ogen.krateo.io
```

If the CRDs do not appear, check the `RestDefinition` status and the
`oasgen-provider` logs — CRD generation is its job, not the chart's.

## Provision a bucket

Bucket provisioning is three CRs, applied in order. Ready-to-edit copies live in the
repo's `samples/` directory.

**1. The credentials Secret** (`samples/secret-credentials.yaml`) — S3 access key in
`username`, secret key in `password`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: hetzner-s3-credentials
  namespace: default
type: Opaque
stringData:
  username: REPLACE_WITH_HETZNER_S3_ACCESS_KEY
  password: REPLACE_WITH_HETZNER_S3_SECRET_KEY
```

**2. The BucketConfiguration** (`samples/bucketconfiguration.yaml`) — references that
Secret through `spec.authentication.basic`:

```yaml
apiVersion: objectstorage.hetzner.ogen.krateo.io/v1alpha1
kind: BucketConfiguration
metadata:
  name: default-fsn1
  namespace: default
spec:
  authentication:
    basic:
      usernameRef:
        name: hetzner-s3-credentials
        namespace: default
        key: username
      passwordRef:
        name: hetzner-s3-credentials
        namespace: default
        key: password
```

**3. The Bucket** (`samples/bucket.yaml`) — the user-facing CR; `spec.name` is the
S3 bucket name, `spec.configurationRef` points at the BucketConfiguration:

```yaml
apiVersion: objectstorage.hetzner.ogen.krateo.io/v1alpha1
kind: Bucket
metadata:
  name: my-krateo-bucket
  namespace: default
spec:
  configurationRef:
    name: default-fsn1
    namespace: default
  name: krateo-demo-bucket-001
  locationConstraint: fsn1
```

Apply them:

```console
$ kubectl apply -f samples/secret-credentials.yaml
$ kubectl apply -f samples/bucketconfiguration.yaml
$ kubectl apply -f samples/bucket.yaml
```

`rest-dynamic-controller` (started by `oasgen-provider`) then runs `findby` on the
first reconcile and `create` if the bucket does not exist, and populates
`status.creationDate` / `status.locationConstraint` on success.

## A note on the data plane

The control plane — the CRDs and the reconcile loop — works. The **data plane**
against a real Hetzner endpoint currently does not, because Hetzner's S3 API needs
AWS SigV4 signing and returns XML, neither of which `rest-dynamic-controller`
supports yet. Expect `403 SignatureDoesNotMatch` against the live endpoint. See
[overview](./overview.md#known-limitations) for the two paths to make it succeed
end-to-end. The CRDs and the user-facing UX are the value this chart delivers today.

## As a Krateo composition

The repo also ships `compositiondefinition.yaml`, which registers this chart with
Krateo as an installable composition (`core.krateo.io/v1alpha1`). Applying it lets
Krateo install the KOG the same way; the per-resource CRs (Secret,
BucketConfiguration, Bucket) are still applied separately from `samples/`. See
[examples](./examples.md).
