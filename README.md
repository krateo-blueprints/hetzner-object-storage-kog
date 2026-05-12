# hetzner-object-storage-kog

A Krateo OASGen operator that provisions Hetzner Object Storage buckets via
[krateoplatformops/oasgen-provider](https://github.com/krateoplatformops/oasgen-provider).

This repository ships:

- `openapi/hetzner-object-storage.yaml` — an OpenAPI 3.0 document describing
  S3 bucket CRUD against `https://{location}.your-objectstorage.com`.
- `chart/` — a Helm chart that installs the OAS as a `ConfigMap` and a
  `RestDefinition` consumed by `oasgen-provider` to generate the `Bucket`
  and `BucketConfiguration` CRDs and start the generic
  `rest-dynamic-controller` against the Hetzner endpoint.
- `samples/` — example user CRs.

## Layout

```
.
├── chart/
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── files/
│   │   └── hetzner-object-storage.yaml      # OAS embedded via .Files.Get
│   └── templates/
│       ├── _helpers.tpl
│       ├── configmap-bucket-oas.yaml        # ConfigMap exposing the OAS
│       └── rd-bucket.yaml                   # RestDefinition for Bucket
├── openapi/
│   └── hetzner-object-storage.yaml          # canonical OAS source
└── samples/
    ├── secret-credentials.yaml              # S3 access key + secret key
    ├── bucketconfiguration.yaml             # per-location config CR
    └── bucket.yaml                          # user-facing Bucket CR
```

## Prerequisites

- A Kubernetes cluster (1.20+).
- `oasgen-provider` installed in the cluster:

  ```sh
  helm repo add krateo https://charts.krateo.io
  helm repo update
  helm install oasgen-provider krateo/oasgen-provider --namespace krateo-system --create-namespace
  ```

- Hetzner S3 credentials (access key + secret key). These can currently only
  be generated from the Hetzner Console — there is no public API.

## Install

From the published OCI artifact (recommended):

```sh
helm install hetzner-object-storage-kog \
  oci://ghcr.io/braghettos/charts/hetzner-object-storage-kog \
  --version 0.1.0 \
  --namespace krateo-system --create-namespace
```

Or from a local checkout:

```sh
helm install hetzner-object-storage-kog ./chart --namespace krateo-system
```

## Releasing

Tag a `v<chart-version>` matching `chart/Chart.yaml`'s `version:` field and
push the tag — the `release-chart` workflow lints, packages, and pushes the
chart to `oci://ghcr.io/braghettos/charts/hetzner-object-storage-kog`.

```sh
git tag v0.1.0
git push origin v0.1.0
```

Once installed, `oasgen-provider` reads the `RestDefinition` and generates:

- `bucket.objectstorage.hetzner.ogen.krateo.io`
- `bucketconfiguration.objectstorage.hetzner.ogen.krateo.io`

## Use

```sh
kubectl apply -f samples/secret-credentials.yaml
kubectl apply -f samples/bucketconfiguration.yaml
kubectl apply -f samples/bucket.yaml
```

The user CR shape:

```yaml
apiVersion: objectstorage.hetzner.ogen.krateo.io/v1alpha1
kind: Bucket
spec:
  configurationRef:
    name: default-fsn1
    namespace: default
  name: krateo-demo-bucket-001
  locationConstraint: fsn1
```

## RestDefinition mapping

| Action  | HTTP Verb | Path        | S3 Operation     |
| ------- | --------- | ----------- | ---------------- |
| findby  | GET       | `/`         | ListBuckets      |
| get     | GET       | `/{name}`   | HeadBucket *(\*)*|
| create  | PUT       | `/{name}`   | CreateBucket     |
| delete  | DELETE    | `/{name}`   | DeleteBucket     |

`findby` is invoked on the first reconcile and matches by bucket `name`;
subsequent reconciles use `get`.

*(\*)* `GET /{name}` is a JSON projection — see Known Limitations.

## Known limitations

This chart deliberately models the S3 bucket REST surface as a JSON OpenAPI
document so that `oasgen-provider` can generate CRDs from it. The Hetzner
Object Storage API is **not** a JSON API, which produces two runtime gaps
that the generic `rest-dynamic-controller` cannot bridge today:

1. **No AWS SigV4 signing.** Every S3 request to Hetzner must carry an
   `Authorization: AWS4-HMAC-SHA256 ...` header computed over the request
   canonicalization. `rest-dynamic-controller` only knows HTTP Basic and
   HTTP Bearer (per the OAS `components.securitySchemes` it accepts). The
   OAS in this chart declares `basicAuth` (S3 access key as username,
   secret key as password) as the closest legal stand-in, but Hetzner will
   reject every request with `403 SignatureDoesNotMatch` until SigV4
   signing is added upstream in the controller (or in a sidecar).

2. **XML responses, not JSON.** S3 returns `ListAllMyBucketsResult`,
   `LocationConstraint`, and `Error` envelopes as XML. The
   `rest-dynamic-controller` parses JSON. Even if signing were solved,
   `findby` and `get` would fail to populate status fields because the
   response body does not match the schemas declared here.

3. **`HEAD /{name}` returns no body.** The S3 idiomatic existence check
   (`HeadBucket`) returns 200 with an empty body. The `get` operation in
   this chart instead documents a JSON projection of `HEAD + GET ?location`
   so that `creationDate` and `locationConstraint` can land in status.
   Against a real Hetzner endpoint that JSON projection does not exist.

4. **`POST /buckets` does not exist on S3.** Bucket creation is `PUT` on
   the bucket path; the spec is modelled this way. Some oasgen-provider
   examples assume `POST` for create — we use `PUT` for fidelity.

### How to actually make this work end-to-end

Two viable paths:

- **Add SigV4 + XML support to `rest-dynamic-controller`** (or a new
  authentication kind in oasgen-provider, e.g. `awsSigV4`). This is the
  most direct upstream fix.
- **Run a small REST→S3 shim service** that exposes the JSON API this
  chart's OAS describes, terminates Basic/Bearer auth, signs SigV4 to
  Hetzner, and translates XML responses to the JSON schemas defined here.
  Point the OAS `servers[0].url` at the shim instead of
  `your-objectstorage.com`. The CRDs and user-facing UX in this chart stay
  unchanged.

Until one of those lands, this chart is useful as a **reference for the
RestDefinition shape and the user-facing CRD contract** for Hetzner
Object Storage, but the data plane will not succeed against the real
endpoint.

## Sources

- oasgen-provider: <https://github.com/krateoplatformops/oasgen-provider>
- Hetzner Object Storage docs: <https://docs.hetzner.com/storage/object-storage/>
- Hetzner Object Storage S3 endpoints: <https://docs.hetzner.com/storage/object-storage/getting-started/using-s3-api-tools/>
- Buckets/objects FAQ (no native JSON API): <https://docs.hetzner.com/storage/object-storage/faq/buckets-objects/>
