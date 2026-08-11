---
type: Runbook
title: hetzner-object-storage-kog — release
description: How a release ships — tag v<chart-version> to lint, package, and push the chart to GHCR via the release-chart workflow, and bump the CompositionDefinition afterwards.
resource: oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog
tags: [krateo, release, oci, ghcr, helm]
timestamp: 2026-08-11T00:00:00Z
---

# Release

A release is one git tag. The tag must be `v<chart-version>` matching
`chart/Chart.yaml`'s `version:` field.

## What a tag ships

The `release-chart` workflow (`.github/workflows/release.yml`) runs on a
`v*.*.*` tag push:

1. **`lint`** — `helm lint chart` plus a `helm template` render smoke test. This job
   also runs on every PR to `main`.
2. **`release`** (tags only) — verifies the tag matches `chart/Chart.yaml`'s
   `version:`, runs `helm package chart`, logs in to GHCR with `GITHUB_TOKEN`, and
   `helm push`es the packaged chart to
   `oci://ghcr.io/<owner>/charts` — i.e.
   `oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog`.

The namespace is derived from `github.repository_owner`, so `GITHUB_TOKEN` only writes
its own org's package namespace.

## Steps

```console
$ git tag v0.1.0
$ git push origin v0.1.0
```

The tag version must equal `chart/Chart.yaml`'s `version:` — the workflow fails the
release with `Git tag v$tag does not match Chart.yaml version` otherwise.

Then verify the artifact exists:

```console
$ helm show chart \
    oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog \
    --version 0.1.0 | head -5
```

## After publishing

Bump `compositiondefinition.yaml`'s `spec.chart.version` to the newly published
version on `main` — it is this component's own Krateo registration and must point at a
chart version that exists. Keep `chart/Chart.yaml`'s `version` and `appVersion` in
step so `krateo.io/*` annotations and the OCI tag line up.

## PR-time checks

On every PR and push to `main`:

- `release-chart`'s `lint` job (`helm lint` + render smoke test).
- `security.yml` — the shared `krateo-platformops/.github` security workflow.
- `lint.yaml` — the shared docs-standard linter
  (`krateo-platformops/.github/.github/workflows/lint-docs.yaml`), which enforces the
  Krateo Documentation Standard on this bundle.

## Downstream version pinning

Consumers install by explicit `--version` ([usage](./usage.md)); nothing tracks a
mutable `latest`. The chart is pushed to an immutable OCI tag per release.
