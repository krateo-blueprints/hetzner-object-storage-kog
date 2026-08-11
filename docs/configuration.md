---
type: Configuration
title: hetzner-object-storage-kog — configuration
description: The whole config surface — the chart's values.yaml (typed by values.schema.json), and the spec fields the Bucket / BucketConfiguration / Secret CRs carry.
resource: oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog
tags: [krateo, oasgen, values, bucket, configuration]
timestamp: 2026-08-11T00:00:00Z
---

# Configuration

There are two layers to configure: the **chart** (a small `values.yaml`, fully typed
by `chart/values.schema.json`) and the **user CRs** the KOG generates
(`Bucket`, `BucketConfiguration`, and the credentials `Secret`).

## Chart values

`chart/values.yaml`, typed by `chart/values.schema.json`:

| key | default | effect |
|---|---|---|
| `nameOverride` | `""` | override the chart name used in resource names. Empty = use the chart name (`hetzner-object-storage-kog`). |
| `fullnameOverride` | `""` | override the fully qualified release name used in resource names. Empty = derive it from the release name + chart name (`_helpers.tpl`). |
| `bucket.oasConfigMapKey` | `hetzner-object-storage.yaml` | the key inside the generated `ConfigMap` that holds the OpenAPI document the `RestDefinition` references. This must match the filename the `RestDefinition`'s `oasPath` points at. |

The surface is intentionally minimal: the chart's job is to publish one OAS
`ConfigMap` and one `RestDefinition`, so there is little to parameterize. The values
schema sets no `additionalProperties`, but every documented key is scalar.

## Derived resource names

From the release name and `_helpers.tpl`, a default install (`helm install
hetzner-object-storage-kog ...`) produces:

- `ConfigMap`: `hetzner-object-storage-kog-bucket-oas`
- `RestDefinition`: `hetzner-object-storage-kog-bucket`

with `spec.oasPath` on the `RestDefinition` resolving to
`configmap://<namespace>/hetzner-object-storage-kog-bucket-oas/hetzner-object-storage.yaml`.

## User CR fields

Once `oasgen-provider` has generated the CRDs, users configure buckets with the
following resources (all in `objectstorage.hetzner.ogen.krateo.io/v1alpha1` except the
Secret). Full CRD shapes are in [api](./api.md); sample manifests are in `samples/`.

### The credentials Secret (`v1`)

| field | required | meaning |
|---|---|---|
| `stringData.username` | yes | Hetzner S3 access key. |
| `stringData.password` | yes | Hetzner S3 secret key. |

The keys are named `username` / `password` because the `BucketConfiguration`
references them by key and the OAS security scheme is HTTP Basic.

### BucketConfiguration

| field | required | meaning |
|---|---|---|
| `spec.authentication.basic.usernameRef` | yes | `{name, namespace, key}` pointing at the Secret key holding the access key. |
| `spec.authentication.basic.passwordRef` | yes | `{name, namespace, key}` pointing at the Secret key holding the secret key. |

A `BucketConfiguration` binds one set of credentials to one endpoint context. Multiple
`Bucket` CRs can share a `BucketConfiguration` via `configurationRef`.

### Bucket

| field | required | meaning |
|---|---|---|
| `spec.configurationRef` | yes | `{name, namespace}` of the `BucketConfiguration` to authenticate with. |
| `spec.name` | yes | the S3 bucket name to create/reconcile. DNS-compatible: 3–63 chars, lowercase, digits, hyphens; globally unique within a Hetzner location. |
| `spec.locationConstraint` | no | Hetzner location code — one of `fsn1`, `nbg1`, `hel1`. |
| `status.creationDate` | (populated) | ISO-8601 creation timestamp — from `additionalStatusFields`. |
| `status.locationConstraint` | (populated) | the effective location — from `additionalStatusFields`. |

`creationDate` is in `excludedSpecFields`, so it never appears in the editable spec —
only in status.

## Endpoint and location

The OAS `servers[0].url` is
`https://{location}.your-objectstorage.com` with `location` defaulting to `fsn1` and
constrained to `fsn1 | nbg1 | hel1`. The effective location on Hetzner is determined
by the endpoint host, so `spec.locationConstraint` on the Bucket is informational — it
lands in status but the host is what routes the request. To target a different
location today you would republish the OAS with a different default (or, in a shim
setup, front it with a location-aware endpoint).

## What is not configurable

There is no value to change the endpoint host, the auth scheme, or the verb mapping
from `values.yaml` — those live in the OAS
(`chart/files/hetzner-object-storage.yaml`) and the `RestDefinition`
(`chart/templates/rd-bucket.yaml`). Changing them means editing those files and
releasing a new chart version ([release](./release.md)).
