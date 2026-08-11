---
type: Architecture
title: hetzner-object-storage-kog — overview
description: What the KOG installs and how it works — the OAS ConfigMap, the RestDefinition, the CRDs oasgen-provider generates, the RestDefinition-to-S3 verb mapping, and the known S3 (SigV4/XML) limitations.
resource: oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog
tags: [krateo, oasgen, restdefinition, s3, hetzner, architecture]
timestamp: 2026-08-11T00:00:00Z
---

# Overview

`hetzner-object-storage-kog` is a **KOG** — a Krateo OASGen operator. It is a Helm
chart that carries an OpenAPI document and a `RestDefinition`, and delegates all
runtime behaviour to
[oasgen-provider](https://github.com/krateo-platformops/oasgen-provider). The chart
never runs a controller of its own; installing it hands `oasgen-provider` a
declarative description of the Hetzner Object Storage bucket API, and the provider
does the rest.

## What the chart installs

One `helm install` renders exactly two manifests (`chart/templates/`):

| manifest | kind | role |
|---|---|---|
| `configmap-bucket-oas.yaml` | `ConfigMap` (`v1`) | holds the OpenAPI 3.0 document (`hetzner-object-storage.yaml`), embedded from `chart/files/` via `.Files.Get`. Named `<fullname>-bucket-oas`. |
| `rd-bucket.yaml` | `RestDefinition` (`ogen.krateo.io/v1alpha1`) | points at that ConfigMap via `spec.oasPath: configmap://<ns>/<fullname>-bucket-oas/hetzner-object-storage.yaml` and declares the resource, identifiers, status fields, and the CRUD verbs. Named `<fullname>-bucket`. |

Both carry the standard `app.kubernetes.io/*` labels from
`chart/templates/_helpers.tpl`. The chart has no other moving parts — the values
surface is small (see [configuration](./configuration.md)).

## How it becomes CRDs

`oasgen-provider` watches `RestDefinition` objects. When it observes the one this
chart installs, it:

1. reads the OAS from the referenced `ConfigMap`;
2. generates the CRDs for the declared `resourceGroup`
   (`objectstorage.hetzner.ogen.krateo.io`):
   - `bucket.objectstorage.hetzner.ogen.krateo.io`
   - `bucketconfiguration.objectstorage.hetzner.ogen.krateo.io`
3. starts a generic `rest-dynamic-controller` instance that reconciles user `Bucket`
   CRs by calling the HTTP verbs declared in the `RestDefinition` against the Hetzner
   endpoint.

So the flow is: **this chart → RestDefinition → oasgen-provider → Bucket /
BucketConfiguration CRDs → rest-dynamic-controller → Hetzner S3**.

## The RestDefinition resource contract

The `RestDefinition` (`chart/templates/rd-bucket.yaml`) declares:

- `resource.kind: Bucket` with `identifiers: [name]` — the field that identifies a
  bucket for `findby`.
- `additionalStatusFields: [creationDate, locationConstraint]` — populated into
  `status` from the API response.
- `excludedSpecFields: [creationDate]` — kept out of the user-editable spec.
- `verbsDescription` — the four CRUD actions and their HTTP verbs and paths.

## RestDefinition-to-S3 verb mapping

| Action | HTTP verb | Path | S3 operation |
|---|---|---|---|
| `findby` | `GET` | `/` | ListBuckets |
| `get` | `GET` | `/{name}` | HeadBucket *(JSON projection)* |
| `create` | `PUT` | `/{name}` | CreateBucket |
| `delete` | `DELETE` | `/{name}` | DeleteBucket |

`findby` runs on the first reconcile and matches by bucket `name`
(`identifiersMatchPolicy: OR`); subsequent reconciles use `get`. Bucket creation is
`PUT` on the bucket path, matching real S3 (not `POST`).

## Authentication model

A user provides Hetzner S3 credentials in a `Secret` (access key as `username`,
secret key as `password`). A `BucketConfiguration` CR references that Secret through
`spec.authentication.basic.usernameRef` / `passwordRef`, and each `Bucket` CR points
at a `BucketConfiguration` via `spec.configurationRef`. The OAS declares `basicAuth`
(`type: http`, `scheme: basic`) as the security scheme, so `rest-dynamic-controller`
sends HTTP Basic — the closest legal stand-in for S3 credentials it can express.

## Known limitations

The Hetzner Object Storage API is S3-compatible: AWS Signature V4 request signing and
XML response envelopes. `rest-dynamic-controller` speaks JSON over HTTP Basic/Bearer.
The OAS in this chart deliberately models the S3 operations in a JSON/Basic shape so
the CRDs can be generated, which leaves runtime gaps the generic controller cannot
bridge today:

1. **No AWS SigV4 signing.** Every S3 request must carry an
   `Authorization: AWS4-HMAC-SHA256 ...` header computed over a canonicalized
   request. The controller only knows HTTP Basic/Bearer, so Hetzner rejects requests
   with `403 SignatureDoesNotMatch`.
2. **XML responses, not JSON.** S3 returns `ListAllMyBucketsResult`,
   `LocationConstraint`, and `Error` as XML; the controller parses JSON, so status
   fields would not populate even if signing were solved.
3. **`HEAD /{name}` returns no body.** The idiomatic `HeadBucket` existence check
   returns `200` with an empty body; the OAS models `get` as a JSON projection of
   `HEAD + GET ?location` so `creationDate` and `locationConstraint` can land in
   status. That projection does not exist on a real endpoint.
4. **`POST /buckets` does not exist on S3.** Create is `PUT` on the bucket path; the
   spec is modelled that way for fidelity.

### Making it work end-to-end

Two viable paths:

- **Add SigV4 + XML support to `rest-dynamic-controller`** (or a new auth kind such as
  `awsSigV4` in oasgen-provider) — the most direct upstream fix.
- **Run a small REST→S3 shim** that exposes the JSON API this OAS describes,
  terminates Basic/Bearer auth, signs SigV4 to Hetzner, and translates XML responses
  to these JSON schemas. Point `servers[0].url` at the shim; the CRDs and user-facing
  UX stay unchanged.

Until one of those lands, this chart is a **reference for the RestDefinition shape and
the user-facing CRD contract** for Hetzner Object Storage; the control plane
(CRD generation and reconcile loop) works, the data plane against the real endpoint
does not.
