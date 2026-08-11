---
type: Log
title: hetzner-object-storage-kog — log
description: Curated chronological history of hetzner-object-storage-kog — notable changes and decisions, not a generated changelog.
resource: oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog
tags: [krateo, oasgen, log, history]
timestamp: 2026-08-11T00:00:00Z
---

# Log

Curated history; release notes live in GitHub Releases.

## 2026-08-11 — Documentation Standard adoption

The repo adopts the Krateo Documentation Standard (OKF): the invariant `docs/` bundle
(`index`, `overview`, `usage`, `configuration`, `api`, `examples`, `release`, `log`
plus `llms.txt`), a runnable `examples/provision-bucket`, a rewritten `README.md` with
the standard six sections, and the shared `lint-docs` check wired into a new
`lint.yaml`. Content is grounded in the real chart, `RestDefinition`, OAS, and
`CompositionDefinition` — including the documented S3 SigV4/XML data-plane limitation.

## 0.1.0 — first release

The KOG ships whole: a Helm chart that installs an OpenAPI 3.0 `ConfigMap` and a
`RestDefinition` (`ogen.krateo.io/v1alpha1`) consumed by
[oasgen-provider](https://github.com/krateo-platformops/oasgen-provider) to generate
the `Bucket` and `BucketConfiguration` CRDs
(`objectstorage.hetzner.ogen.krateo.io/v1alpha1`) and start the generic
`rest-dynamic-controller` against a Hetzner S3 endpoint. Two decisions worth keeping:

- **The S3 surface is modelled as a JSON OpenAPI document** so `oasgen-provider` can
  generate CRDs from it. This is deliberate, and it makes the control plane (CRD
  generation, reconcile loop, user-facing CRD contract) work today.
- **The data plane against a real endpoint does not yet succeed.** Hetzner's S3 API
  needs AWS SigV4 signing and returns XML; `rest-dynamic-controller` speaks JSON over
  HTTP Basic/Bearer. The chart documents this gap and the two ways to close it
  (upstream SigV4/XML support, or a REST→S3 shim), and stands as the reference
  RestDefinition shape and CRD contract for Hetzner Object Storage in the meantime.

`create` is `PUT /{name}` (matching real S3), not `POST`; `findby` is `GET /` matched
by bucket `name`; `creationDate` and `locationConstraint` are projected into status.
