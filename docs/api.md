---
type: API
title: hetzner-object-storage-kog — API
description: The resources this KOG exposes — the RestDefinition the chart emits, the Bucket and BucketConfiguration CRDs oasgen-provider generates from it, the underlying OpenAPI operations, and the CompositionDefinition registration.
resource: oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog
tags: [krateo, oasgen, restdefinition, crd, api, s3]
timestamp: 2026-08-11T00:00:00Z
---

# API

This KOG exposes three API layers: the `RestDefinition` the chart installs, the CRDs
`oasgen-provider` generates from it, and the OpenAPI operations those CRDs ultimately
drive. It also ships a `CompositionDefinition` that registers the chart with Krateo.

## The RestDefinition the chart emits

`chart/templates/rd-bucket.yaml`, `ogen.krateo.io/v1alpha1`, kind `RestDefinition`,
named `<fullname>-bucket`:

```yaml
apiVersion: ogen.krateo.io/v1alpha1
kind: RestDefinition
metadata:
  name: hetzner-object-storage-kog-bucket
  namespace: krateo-system
spec:
  oasPath: configmap://krateo-system/hetzner-object-storage-kog-bucket-oas/hetzner-object-storage.yaml
  resourceGroup: objectstorage.hetzner.ogen.krateo.io
  resource:
    kind: Bucket
    identifiers:
      - name
    additionalStatusFields:
      - creationDate
      - locationConstraint
    excludedSpecFields:
      - creationDate
    verbsDescription:
      - action: findby
        method: GET
        path: /
        identifiersMatchPolicy: OR
      - action: get
        method: GET
        path: /{name}
      - action: create
        method: PUT
        path: /{name}
      - action: delete
        method: DELETE
        path: /{name}
```

| field | value | meaning |
|---|---|---|
| `spec.oasPath` | `configmap://<ns>/<fullname>-bucket-oas/hetzner-object-storage.yaml` | where oasgen-provider reads the OAS (the ConfigMap this chart also installs). |
| `spec.resourceGroup` | `objectstorage.hetzner.ogen.krateo.io` | the API group of the generated CRDs. |
| `resource.kind` | `Bucket` | the generated resource kind. |
| `resource.identifiers` | `[name]` | the field used to identify a bucket for `findby`. |
| `resource.additionalStatusFields` | `[creationDate, locationConstraint]` | response fields projected into `status`. |
| `resource.excludedSpecFields` | `[creationDate]` | fields kept out of the editable spec. |
| `resource.verbsDescription` | four actions | the CRUD verbs (see the mapping below). |

## CRDs oasgen-provider generates

Reading that `RestDefinition`, `oasgen-provider` generates two CRDs in the
`objectstorage.hetzner.ogen.krateo.io` group, `v1alpha1`:

### `bucket.objectstorage.hetzner.ogen.krateo.io`

Kind `Bucket`. Spec (from the OAS `Bucket` / `CreateBucketConfiguration` schemas,
minus `excludedSpecFields`):

| field | type | required | meaning |
|---|---|---|---|
| `spec.configurationRef.name` | string | yes | name of the `BucketConfiguration` to authenticate with. |
| `spec.configurationRef.namespace` | string | yes | its namespace. |
| `spec.name` | string | yes | the S3 bucket name (DNS-compatible, 3–63 chars, lowercase). |
| `spec.locationConstraint` | string (`fsn1`\|`nbg1`\|`hel1`) | no | Hetzner location code. |
| `status.creationDate` | string | — | ISO-8601 creation timestamp (projected from the API). |
| `status.locationConstraint` | string | — | effective location (projected from the API). |

### `bucketconfiguration.objectstorage.hetzner.ogen.krateo.io`

Kind `BucketConfiguration`. Binds credentials to the endpoint:

| field | type | required | meaning |
|---|---|---|---|
| `spec.authentication.basic.usernameRef` | `{name, namespace, key}` | yes | Secret reference for the S3 access key. |
| `spec.authentication.basic.passwordRef` | `{name, namespace, key}` | yes | Secret reference for the S3 secret key. |

The credentials themselves live in a plain `Secret` (`v1`, `stringData.username` /
`stringData.password`).

## The underlying OpenAPI operations

The OAS (`openapi/hetzner-object-storage.yaml`, embedded at
`chart/files/hetzner-object-storage.yaml`) describes the S3 bucket surface as a JSON
API. `rest-dynamic-controller` calls these per the `verbsDescription`:

| operationId | verb / path | RestDefinition action | S3 operation | responses |
|---|---|---|---|---|
| `listBuckets` | `GET /` | `findby` | ListBuckets | 200 `ListAllMyBucketsResult`, 403 |
| `getBucket` | `GET /{name}` | `get` | HeadBucket (JSON projection) | 200 `Bucket`, 404, 403 |
| `createBucket` | `PUT /{name}` | `create` | CreateBucket | 200 `Bucket`, 409, 403 |
| `deleteBucket` | `DELETE /{name}` | `delete` | DeleteBucket | 204, 404, 409, 403 |

Security scheme: `basicAuth` (`type: http`, `scheme: basic`) — S3 access key as
username, secret key as password. Server:
`https://{location}.your-objectstorage.com`, `location ∈ {fsn1, nbg1, hel1}`,
default `fsn1`.

Key schemas: `Bucket` (`name`, `creationDate`, `locationConstraint`),
`CreateBucketConfiguration` (`locationConstraint`),
`ListAllMyBucketsResult` (`owner`, `buckets[]`), and `Error`
(`code`, `message`, `bucketName`, `requestId`) — all JSON projections of the
corresponding S3 XML envelopes.

## The CompositionDefinition registration

`compositiondefinition.yaml` registers the chart with Krateo as an installable
composition:

```yaml
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: hetzner-object-storage-kog
  namespace: krateo-system
spec:
  chart:
    url: oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog
    version: "0.1.0"
```

`spec.chart.version` must be a published chart version. Krateo derives the composition
kind from the chart name and version — `hetzner-object-storage-kog` →
`HetznerObjectStorageKog`, `0.1.0` → `composition.krateo.io/v0-1-0` (the shape used in
`examples/composition.yaml`). Installing the composition installs the KOG; the
per-resource CRs (Secret, BucketConfiguration, Bucket) are applied separately.

## Runtime caveat

These CRDs are generated and reconciled correctly, but calls to a real Hetzner
endpoint fail because the S3 API needs AWS SigV4 signing and returns XML, neither of
which `rest-dynamic-controller` supports yet. See
[overview](./overview.md#known-limitations).
