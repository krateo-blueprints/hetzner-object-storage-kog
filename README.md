# hetzner-object-storage-kog

A Krateo OASGen operator (KOG) that provisions Hetzner Object Storage buckets: a Helm
chart that installs an OpenAPI document and a `RestDefinition` so
[oasgen-provider](https://github.com/krateo-platformops/oasgen-provider) generates the
`Bucket` and `BucketConfiguration` CRDs and drives S3 bucket CRUD against a Hetzner
endpoint.

## What is this

This repository ships a **KOG** — it does not run a controller itself, it hands
`oasgen-provider` a declarative description of the Hetzner Object Storage bucket API:

- `openapi/hetzner-object-storage.yaml` — an OpenAPI 3.0 document describing S3 bucket
  CRUD against `https://{location}.your-objectstorage.com` (canonical source; a copy
  is embedded at `chart/files/`).
- `chart/` — a Helm chart that installs the OAS as a `ConfigMap`
  (`chart/templates/configmap-bucket-oas.yaml`) and a `RestDefinition`
  (`chart/templates/rd-bucket.yaml`, `ogen.krateo.io/v1alpha1`). `oasgen-provider`
  reads the `RestDefinition` and generates the CRDs
  `bucket.objectstorage.hetzner.ogen.krateo.io` and
  `bucketconfiguration.objectstorage.hetzner.ogen.krateo.io`, then starts the generic
  `rest-dynamic-controller`.
- `samples/` — ready-to-edit user CRs (the credentials `Secret`, the
  `BucketConfiguration`, and the `Bucket`).
- `compositiondefinition.yaml` — an optional Krateo `CompositionDefinition` that
  registers the chart as an installable composition.

> **Data-plane caveat.** Hetzner's S3 API needs AWS SigV4 signing and returns XML,
> while `rest-dynamic-controller` speaks JSON over HTTP Basic/Bearer. The control
> plane (CRD generation and the reconcile loop) works; calls to a real endpoint fail
> with `403 SignatureDoesNotMatch` until upstream SigV4/XML support (or a REST→S3
> shim) is added. See [docs/overview.md](./docs/overview.md#known-limitations).

## Install

Prerequisite: `oasgen-provider` must already be installed in the cluster.

```sh
helm repo add krateo https://charts.krateo.io
helm repo update
helm install oasgen-provider krateo/oasgen-provider \
  --namespace krateo-system --create-namespace
```

Then install this KOG from the published OCI artifact (recommended):

```sh
helm install hetzner-object-storage-kog \
  oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog \
  --version 0.1.0 \
  --namespace krateo-system --create-namespace
```

Or from a local checkout:

```sh
helm install hetzner-object-storage-kog ./chart --namespace krateo-system
```

Full walk-through: [docs/usage.md](./docs/usage.md).

## Configure

The chart's `values.yaml` (typed by `chart/values.schema.json`) is minimal:

| key | default | effect |
|---|---|---|
| `nameOverride` | `""` | override the chart name used in resource names. |
| `fullnameOverride` | `""` | override the fully qualified release name. |
| `bucket.oasConfigMapKey` | `hetzner-object-storage.yaml` | the key in the OAS `ConfigMap` the `RestDefinition` references. |

The buckets themselves are configured with the `Bucket`, `BucketConfiguration`, and
credentials `Secret` CRs. Full surface: [docs/configuration.md](./docs/configuration.md).

## Examples

- [examples/provision-bucket](./examples/provision-bucket/README.md) — provision a
  bucket end to end (install, then Secret → BucketConfiguration → Bucket).
- `examples/composition.yaml` — install the KOG through Krateo as a
  `HetznerObjectStorageKog` composition.

Index: [docs/examples.md](./docs/examples.md).

## Docs

- [docs/index.md](./docs/index.md) — the map of the bundle.
- [docs/overview.md](./docs/overview.md) — architecture and known limitations.
- [docs/usage.md](./docs/usage.md) — install and provision.
- [docs/configuration.md](./docs/configuration.md) — the whole config surface.
- [docs/api.md](./docs/api.md) — the RestDefinition, the generated CRDs, and the OAS.
- [docs/examples.md](./docs/examples.md) — runnable examples.
- [docs/release.md](./docs/release.md) — how a release ships.
- [docs/log.md](./docs/log.md) — curated history.
- [docs/llms.txt](./docs/llms.txt) — the LLM doc index.

## Develop & release

Tag `v<chart-version>` matching `chart/Chart.yaml`'s `version:` and push it — the
`release-chart` workflow (`.github/workflows/release-tag.yaml`) lints, packages, and pushes
the chart to `oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog`.

```sh
git tag v0.1.0
git push origin v0.1.0
```

After publishing, bump `compositiondefinition.yaml`'s `spec.chart.version` on `main`.
CI runs `helm lint` + a render smoke test, the shared security workflow, and the
shared docs-standard linter (`.github/workflows/lint.yaml`). Full runbook:
[docs/release.md](./docs/release.md).

## Sources

- oasgen-provider: <https://github.com/krateo-platformops/oasgen-provider>
- Hetzner Object Storage docs: <https://docs.hetzner.com/storage/object-storage/>
- Hetzner S3 API tooling: <https://docs.hetzner.com/storage/object-storage/getting-started/using-s3-api-tools/>
