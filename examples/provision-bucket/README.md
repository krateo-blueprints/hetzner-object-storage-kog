---
type: Example
title: provision-bucket — provision a Hetzner Object Storage bucket
description: End-to-end walk-through — install oasgen-provider, install the KOG, then apply the credentials Secret, a BucketConfiguration, and a Bucket CR to provision a bucket via the generated CRDs.
resource: oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog
tags: [example, krateo, oasgen, bucket, hetzner]
timestamp: 2026-08-11T00:00:00Z
---

# provision-bucket

Provision a Hetzner Object Storage bucket end to end with this KOG. The manifests
used here are the ready-to-edit ones in the repo's `samples/` directory:
`secret-credentials.yaml`, `bucketconfiguration.yaml`, and `bucket.yaml`.

## Prerequisites

- A Kubernetes cluster (1.20+).
- [`oasgen-provider`](https://github.com/krateo-platformops/oasgen-provider) installed:

  ```console
  $ helm repo add krateo https://charts.krateo.io
  $ helm repo update
  $ helm install oasgen-provider krateo/oasgen-provider \
      --namespace krateo-system --create-namespace
  ```

- Hetzner S3 credentials (access key + secret key), generated from the Hetzner
  Console.

## 1. Install the KOG

```console
$ helm install hetzner-object-storage-kog \
    oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog \
    --version 0.1.0 \
    --namespace krateo-system --create-namespace
```

This installs the OAS `ConfigMap` and the `RestDefinition`. Confirm
`oasgen-provider` generated the CRDs:

```console
$ kubectl get crd | grep objectstorage.hetzner.ogen.krateo.io
bucket.objectstorage.hetzner.ogen.krateo.io
bucketconfiguration.objectstorage.hetzner.ogen.krateo.io
```

> Alternative entry point: instead of `helm install`, apply
> `examples/composition.yaml` (a `HetznerObjectStorageKog` composition CR) to install
> the KOG through Krateo. The steps below are identical afterwards.

## 2. Create the credentials Secret

Edit `samples/secret-credentials.yaml` and replace the placeholders with your Hetzner
S3 access key (`username`) and secret key (`password`), then apply it:

```console
$ kubectl apply -f samples/secret-credentials.yaml
```

## 3. Create the BucketConfiguration

`samples/bucketconfiguration.yaml` references the Secret by key. Apply it as-is (or
adjust names/namespaces to match your Secret):

```console
$ kubectl apply -f samples/bucketconfiguration.yaml
```

## 4. Create the Bucket

`samples/bucket.yaml` points at the `BucketConfiguration` and sets the S3 bucket name
and location:

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

```console
$ kubectl apply -f samples/bucket.yaml
```

## 5. Inspect the reconcile

```console
$ kubectl get bucket -n default
$ kubectl describe bucket my-krateo-bucket -n default
```

On success, `status.creationDate` and `status.locationConstraint` are populated
(projected from the API response by the `additionalStatusFields` in the
`RestDefinition`).

## Expected outcome and the caveat

The **control plane** works: the CRDs are generated and the `Bucket` CR is reconciled
by `rest-dynamic-controller`. Against a **real** Hetzner endpoint the **data plane**
call currently fails with `403 SignatureDoesNotMatch`, because Hetzner's S3 API
requires AWS SigV4 signing and returns XML, which `rest-dynamic-controller` does not
yet support. This example demonstrates the full CRD contract and reconcile flow; see
[../../docs/overview.md](../../docs/overview.md#known-limitations) for the two paths to
make the data plane succeed (upstream SigV4/XML support, or a REST→S3 shim).

## Clean up

```console
$ kubectl delete -f samples/bucket.yaml
$ kubectl delete -f samples/bucketconfiguration.yaml
$ kubectl delete -f samples/secret-credentials.yaml
$ helm uninstall hetzner-object-storage-kog --namespace krateo-system
```
