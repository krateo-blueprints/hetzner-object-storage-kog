---
type: Component
title: hetzner-object-storage-kog — index
description: The map of the hetzner-object-storage-kog doc bundle — a Krateo OASGen operator (KOG) that installs a RestDefinition so oasgen-provider generates the Bucket and BucketConfiguration CRDs against a Hetzner Object Storage S3 endpoint.
resource: oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog
tags: [krateo, oasgen, kog, storage, s3, hetzner]
timestamp: 2026-08-11T00:00:00Z
---

# hetzner-object-storage-kog

`hetzner-object-storage-kog` is a **Krateo OASGen operator (KOG)**: a Helm chart
that ships an OpenAPI 3.0 document for the Hetzner Object Storage S3 bucket surface
and a `RestDefinition` (`ogen.krateo.io/v1alpha1`). Once installed,
[oasgen-provider](https://github.com/krateo-platformops/oasgen-provider) reads that
`RestDefinition`, generates the `Bucket` and `BucketConfiguration` CRDs, and starts
the generic `rest-dynamic-controller` to drive S3 bucket CRUD against a Hetzner
endpoint (`https://{location}.your-objectstorage.com`).

The chart itself deploys no controller of its own — it is a declarative wrapper: an
OAS-carrying `ConfigMap` plus one `RestDefinition`. All the runtime behaviour comes
from `oasgen-provider`, which must already be present in the cluster.

## The bundle (start here)

- [overview](./overview.md) — what a KOG is, the resources this chart installs, the
  generated CRDs, and the RestDefinition-to-S3 verb mapping.
- [usage](./usage.md) — install `oasgen-provider`, install this chart, and provision
  a bucket (Secret → BucketConfiguration → Bucket).
- [configuration](./configuration.md) — the whole values surface plus the CRD spec
  fields the user CRs carry.
- [api](./api.md) — the `RestDefinition` the chart emits and the generated `Bucket` /
  `BucketConfiguration` CRDs, with their group/version and fields.
- [examples](./examples.md) — the runnable example under `examples/`.
- [release](./release.md) — how a tag ships the chart to GHCR.
- [log](./log.md) — curated history.
- [llms.txt](./llms.txt) — the doc index of this bundle.

## Layout

- `chart/` — the KOG chart:
  - `files/hetzner-object-storage.yaml` — the OAS embedded via `.Files.Get`.
  - `templates/configmap-bucket-oas.yaml` — the `ConfigMap` exposing that OAS.
  - `templates/rd-bucket.yaml` — the `RestDefinition` for the `Bucket` resource.
  - `values.yaml` + `values.schema.json` — the (small) config surface.
- `openapi/hetzner-object-storage.yaml` — the canonical OAS source (embedded copy
  lives under `chart/files/`).
- `samples/` — ready-to-edit user CRs: the S3 credentials `Secret`, the
  `BucketConfiguration`, and the `Bucket`.
- `examples/provision-bucket/` — a walk-through example built from those samples.
- `compositiondefinition.yaml` — an optional `CompositionDefinition` that registers
  this chart with Krateo as an installable composition.

## Known limitations

The Hetzner Object Storage API is S3-compatible (AWS Signature V4 + XML), not a JSON
API, while `rest-dynamic-controller` speaks JSON over HTTP Basic/Bearer. The OAS in
this chart models the S3 operations in a JSON/Basic shape so the CRDs can be
generated, but the data plane will not succeed against a real Hetzner endpoint until
SigV4 signing and XML handling are added upstream (or a REST→S3 shim is placed in
front). See [overview](./overview.md#known-limitations) for the full breakdown. This
chart is production-useful today as the **reference RestDefinition shape and the
user-facing CRD contract** for Hetzner Object Storage.
