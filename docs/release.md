---
type: Runbook
title: hetzner-object-storage-kog — release
description: How a release ships — tag a bare semver to push the chart to GHCR and publish the same archive to the Marketplace's blueprints index, per the organisation's publishing standard.
resource: oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog
tags: [krateo, release, oci, ghcr, helm]
timestamp: 2026-11-07T00:00:00Z
---

# Release

A release is one git tag: a bare semver such as `0.1.3` (no `v` prefix). The committed
`chart/Chart.yaml` and `compositiondefinition.yaml` carry the `CHART_VERSION` placeholder;
the release stamps both from the tag, so neither is ever bumped by hand.

## What a tag ships

`.github/workflows/release-tag.yaml` runs on the tag and publishes the chart twice, with the
same bytes, as the organisation's publishing standard requires
([krateo-blueprints/charts PUBLISHING.md](https://github.com/krateo-blueprints/charts/blob/main/PUBLISHING.md)):

1. **Refuses to start** unless the tagged commit has a successful `lint` run and the org
   secret `CHARTS_PUBLISH_TOKEN` is set, so nothing reaches GHCR that cannot also reach
   the Marketplace.
2. **Packages and checks** the chart. It is a blueprint (it has a `values.schema.json` and
   a `CompositionDefinition`), so the stamped `compositiondefinition.yaml` is copied into
   the chart root before `helm package`, and the archive itself is checked: it must hold
   `compositiondefinition.yaml` (installing
   `oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog` at the tag's
   version) and `values.schema.json`, and `Chart.yaml` must have an https `icon` and
   `keywords`.
3. **Pushes** it to `oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog:<tag>`.
4. **Publishes** the same archive to the Marketplace's `blueprints` index through the
   organisation's publisher (`krateo-blueprints/charts/.github/workflows/publish-chart.yaml`, pinned per its rule 9),
   which succeeds only once the served index lists the version and `helm pull --repo`
   installs it anonymously. The portal Marketplace installs from that index, not from GHCR.
5. **Attaches** the stamped `compositiondefinition.yaml` to the GitHub release for this tag.

## Steps

```console
$ git tag 0.1.3
$ git push origin 0.1.3
```

Then verify both copies:

```console
$ helm show chart \
    oci://ghcr.io/krateo-blueprints/charts/hetzner-object-storage-kog \
    --version 0.1.3 | head -5
$ helm pull hetzner-object-storage-kog --version 0.1.3 \
    --repo https://krateo-blueprints.github.io/charts/blueprints
```

## PR-time checks

On every PR and push to `main`:

- `lint.yaml` — the shared docs-standard linter
  (`krateo-platformops/.github/.github/workflows/lint-docs.yaml`) plus an informational
  `helm lint` / `helm template` of the chart. Its run on the tagged commit is what the
  release requires.
- `security.yml` — the shared `krateo-platformops/.github` security workflow.
- `preflight-refs.yaml` — resolves every chart and image reference against its registry.

## Downstream version pinning

Consumers install by explicit `--version` ([usage](./usage.md)); nothing tracks a
mutable `latest`. The chart is pushed to an immutable OCI tag per release.
