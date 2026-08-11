---
type: ExampleIndex
title: hetzner-object-storage-kog — examples
description: Index of the runnable examples under examples/ — provisioning a Hetzner Object Storage bucket end to end.
resource: oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog
tags: [krateo, oasgen, examples, bucket, hetzner]
timestamp: 2026-08-11T00:00:00Z
---

# Examples

- [examples/provision-bucket](../examples/provision-bucket/README.md) — the full
  provisioning walk-through: install `oasgen-provider`, install this KOG, apply the
  credentials `Secret`, the `BucketConfiguration`, and the `Bucket` CR, and check the
  generated CRDs and the reconcile status. Uses the ready-to-edit manifests in the
  repo's `samples/` directory.

The repo also carries `examples/composition.yaml` — the `CompositionDefinition`-style
`HetznerObjectStorageKog` composition CR that installs the KOG through Krateo instead
of a direct `helm install`. The `provision-bucket` example covers both entry points.
